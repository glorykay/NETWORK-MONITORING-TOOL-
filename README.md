# NETWORK-MONITORING-TOOL-
This is a Network monitoring tool likely bmon, nload show the real time monitoring
#!/usr/bin/env bash
# world_map_monitor.sh
# Draws ASCII world map in blue/green with header "WORD MAP"
# Continuously shows network traffic (connections summary by default, or live packets with --packets)
# WARNING: Read-only monitoring only. Use on machines/networks you own or are authorized to inspect.

set -euo pipefail

# Colors
ESC="\e"
RESET="${ESC}[0m"
BOLD="${ESC}[1m"
BLUE_BG="${ESC}[44m${ESC}[37m"   # blue background (ocean) + white text
BLUE_FG="${ESC}[34m"
GREEN_FG="${ESC}[32m"
RED_FG="${ESC}[31m"
CYAN_FG="${ESC}[36m"
YELLOW_FG="${ESC}[33m"

# Map style characters
OCEAN_CHAR="░"
LAND_CHAR="█"

MODE="connections"
if [ "${1:-}" = "--packets" ] || [ "${2:-}" = "--packets" ]; then
  MODE="packets"
fi

# Terminal size detection (fallbacks)
TERM_COLS=$(tput cols 2>/dev/null || echo 80)
TERM_LINES=$(tput lines 2>/dev/null || echo 24)

# Small ASCII "world map" matrix (very approximate and stylized)
# Each line is a mix of ocean and land characters; we color them.
MAP_LINES=(
"      ${LAND_CHAR}${LAND_CHAR}${LAND_CHAR}          ${LAND_CHAR}${LAND_CHAR}          ${LAND_CHAR}${LAND_CHAR}${LAND_CHAR}      "
"    ${LAND_CHAR}${LAND_CHAR}${LAND_CHAR}${LAND_CHAR}${LAND_CHAR}     ${LAND_CHAR}   ${LAND_CHAR}   ${LAND_CHAR}${LAND_CHAR}${LAND_CHAR}${LAND_CHAR}  "
"   ${LAND_CHAR}${LAND_CHAR}   ${LAND_CHAR}    ${LAND_CHAR}                ${LAND_CHAR}    ${LAND_CHAR}${LAND_CHAR} "
"   ${LAND_CHAR}         ${LAND_CHAR}   ${LAND_CHAR}${LAND_CHAR}${LAND_CHAR}   ${LAND_CHAR}      ${LAND_CHAR}     "
"    ${LAND_CHAR}       ${LAND_CHAR}    ${LAND_CHAR}   ${LAND_CHAR}    ${LAND_CHAR}       "
"        ${LAND_CHAR}${LAND_CHAR}${LAND_CHAR}         ${LAND_CHAR}${LAND_CHAR}           ${LAND_CHAR}    "
"           ${LAND_CHAR}              ${LAND_CHAR}${LAND_CHAR}           "
)

# Print header and the map
print_map() {
  # clear the screen
  printf "%b" "${ESC}[2J${ESC}[H"

  # Header: user requested text exactly as "WORD MAP"
  printf "%b" "${BOLD}${RED_FG}$(printf '  %-'"$((TERM_COLS-2))" " " | sed 's/ / /g')${RESET}\n"
  printf "%b" "${BOLD}${RED_FG}  WORD MAP${RESET}\n\n"

  # Center the map horizontally
  map_width=${#MAP_LINES[0]}
  left_pad=$(( (TERM_COLS - map_width) / 2 ))
  [ "$left_pad" -lt 0 ] && left_pad=0
  pad="$(printf '%*s' "$left_pad")"

  # Print each map line with colored ocean/land
  for ln in "${MAP_LINES[@]}"; do
    # Replace LAND_CHAR with green block, OCEAN_CHAR with blue block
    # Our MAP_LINES has explicit LAND_CHAR and spaces for ocean; we'll render spaces as ocean
    line_render="$ln"
    # Convert spaces to ocean char for nicer look
    line_render="${line_render// /$OCEAN_CHAR}"

    # Now color characters: LAND_CHAR -> green BLOCK, OCEAN_CHAR -> blue BLOCK
    # Use printf with escape codes; iterate chars to avoid expansion issues
    printf "%s" "$pad"
    for (( i=0; i<${#line_render}; i++ )); do
      ch="${line_render:i:1}"
      if [ "$ch" = "$LAND_CHAR" ]; then
        printf "%b" "${GREEN_FG}${ch}${RESET}"
      else
        printf "%b" "${BLUE_FG}${ch}${RESET}"
      fi
    done
    printf "\n"
  done

  printf "\n"
}

# Print a small footer with mode info and usage
print_footer() {
  echo -e "${CYAN_FG}Mode:${RESET} ${BOLD}${MODE}${RESET}    ${CYAN_FG}Tips:${RESET} Ctrl-C to quit. Use --packets for live packet stream (requires sudo)."
  echo
}

# Show connection summary continuously (non-destructive)
stream_connections() {
  # We'll refresh the map + connections every second.
  # On each iteration, print the map and then a snapshot of ss (active connections).
  while true; do
    print_map
    print_footer

    echo -e "${YELLOW_FG}${BOLD}Active listening sockets and established connections (snapshot):${RESET}"
    if command -v ss >/dev/null 2>&1; then
      ss -tunap state established,listen 2>/dev/null | sed -n '1,200p' || ss -tunp 2>/dev/null | sed -n '1,200p'
    elif command -v netstat >/dev/null 2>&1; then
      netstat -tulpn 2>/dev/null | sed -n '1,200p'
    else
      echo "(Neither ss nor netstat available.)"
    fi

    echo
    echo -e "${YELLOW_FG}${BOLD}Top processes by network bytes (recent):${RESET}"
    # Use /proc/net/dev to show interface bytes delta over 1 second
    if [ -r /proc/net/dev ]; then
      # sample bytes, sleep, sample again, compute delta
      awk '/:/ {gsub(":",""); print $1, $2, $10}' /proc/net/dev > /tmp/_netdev_before.$$ || true
      sleep 1
      awk '/:/ {gsub(":",""); print $1, $2, $10}' /proc/net/dev > /tmp/_netdev_after.$$ || true
      awk 'NR==FNR{b[$1]=$2 FS $3; next}{ split(b[$1],a," "); inb=a[1]; inb2=a[2]; out=$2; out2=$3; d_in=out-inb; d_out=out2-inb2; printf "%-10s IN: %10d  OUT: %10d\n",$1,d_in,d_out}' /tmp/_netdev_before.$$ /tmp/_netdev_after.$$ | sed -n '1,100p'
      rm -f /tmp/_netdev_before.$$ /tmp/_netdev_after.$$
    else
      echo "(Can't read /proc/net/dev)"
    fi

    # small delay before refreshing; keep map at top of terminal
    sleep 1
  done
}

# Stream raw packets under the map (tcpdump)
stream_packets() {
  # tcpdump will run and print packets to stdout line-by-line. We'll print the map once
  # and then let tcpdump stream output beneath it. If tcpdump isn't present, show error.
  print_map
  print_footer

  if ! command -v tcpdump >/dev/null 2>&1; then
    echo -e "${RED_FG}tcpdump is not installed. Install it (e.g., apt install tcpdump) and re-run with sudo if you want packet streaming.${RESET}"
    exit 1
  fi

  echo -e "${YELLOW_FG}${BOLD}Streaming packets (tcpdump) - press Ctrl-C to stop.${RESET}"
  echo -e "${YELLOW_FG}Note: tcpdump requires elevated privileges to capture traffic on many systems.${RESET}"
  echo

  # Run tcpdump line-buffered so output flows properly.
  # We use -n (no DNS), -l (line-buffered), -i any (all interfaces), and -vv for detail.
  # User can interrupt with Ctrl-C.
  if [ "$(id -u)" -ne 0 ]; then
    if command -v sudo >/dev/null 2>&1; then
      exec sudo tcpdump -n -l -i any -vv
    else
      echo "(Not root and sudo not found; cannot run tcpdump.)"
      exit 1
    fi
  else
    exec tcpdump -n -l -i any -vv
  fi
}

# Handle SIGINT to exit cleanly
trap 'printf "\n\n${BOLD}${RED_FG}Exiting world_map_monitor...${RESET}\n"; exit 0' INT TERM

# Entrypoint
if [ "$MODE" = "connections" ]; then
  stream_connections
else
  # packets mode
  stream_packets
fi
