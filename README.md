# ADB_SCRIPT.LOG-Automated_response-against-unicode-commands-attack-surface

#!/bin/bash

# Configuration
LOG_FILE="adb_session.log"
DEVICE_SERIAL="your_device_serial" # Replace
PROBABILITY_FLAG="0.1"
TIMEOUT_VERIFY="3"
TIMEOUT_ADB="5"
SLEEP_TIME="0.5"
INPUT_SOURCE="auto" # "auto" or "string"
ENABLE_UNICODE_DEFENSE="true"
ENABLE_COMMAND_VERIFY="true"
ENABLE_SYSTEM_CHECKS="true"
ENABLE_ADAPTIVE_BEHAVIOR="true"

# Logging
log_message() {
    local message="$1"
    date +"%Y-%m-%d %H:%M:%S" "$message" >> "$LOG_FILE"
    echo "$(date +"%Y-%m-%d %H:%M:%S") $message"
}

# System Checks
check_adb_binary() {
    if ! command -v adb &> /dev/null; then
        log_message "ERROR: adb binary not found."
        return 1
    fi
    return 0
}

check_device_connection() {
    local device_check=$(adb devices | grep "$DEVICE_SERIAL" | awk '{print $2}')
    if [[ -z "$device_check" || "$device_check" != "device" ]]; then
        log_message "ERROR: Device not connected or unauthorized."
        return 1
    fi
    return 0
}

# Unicode Sanitization
sanitize_unicode_adb() {
    local input_text="$1"
    local sanitized_text=$(echo "$input_text" | sed 's/[\x00-\x1F\x7F-\x9F]//g' | sed 'y/\u0430\u0435\u043A/aek/')
    sanitized_text=$(echo "$sanitized_text" | sed 's/\\/\\\\/g; s/"/\\"/g; s/'"'/\\'/g; s/;/\\;/g' | tr -dc '[:print:]')
    echo "$sanitized_text"
}

sanitize_unicode_adb_defense() {
    local input_text="$1"
    local sanitized_text=$(sanitize_unicode_adb "$input_text")
    sanitized_text=$(echo "$sanitized_text" | sed 's/[\u202E\u202D\u200E\u200F]//g')
    sanitized_text=$(echo "$sanitized_text" | sed 's/[\u200B\u200C\u200D]//g')
    log_message "Unicode Defense Sanitized: $input_text -> $sanitized_text"
    echo "$sanitized_text"
}

# Command Verification
verify_adb_command() {
    local command="$1"
    local temp_dir=$(mktemp -d)
    if [[ -z "$temp_dir" ]]; then
        log_message "ERROR: Failed to create temporary directory."
        return 1
    fi
    local result=$(timeout "$TIMEOUT_VERIFY" bash -c "cd \"$temp_dir\"; $command" 2>/dev/null)
    local return_code=$?
    rm -rf "$temp_dir"
    if [[ "$return_code" -ne 0 ]]; then
        log_message "Command verification failed: $command"
        return 1
    fi
    return 0
}

verify_adb_command_defense() {
    local command="$1"
    local result=$(verify_adb_command "$command")
    if [[ "$result" -ne 0 ]]; then
      return 1
    fi
    local random_value=$(od -vAn -N4 -tu4 /dev/urandom | awk '{print $1}')
    local probability_check=$((random_value % 100))
    if [[ "$probability_check" -lt $(echo "$PROBABILITY_FLAG * 100" | bc | awk '{print int($1)}') ]]; then
        log_message "Probabilistic verification flagged: $command"
        return 1
    fi
    return 0
}

# ADB Command Execution
send_adb_command() {
    local input_text="$1"
    local sanitized_text
    if [[ "$ENABLE_UNICODE_DEFENSE" == "true" ]]; then
        sanitized_text=$(sanitize_unicode_adb_defense "$input_text")
    else
        sanitized_text=$(sanitize_unicode_adb "$input_text")
    fi
    local command="adb -s \"$DEVICE_SERIAL\" shell \"$sanitized_text\""
    local verify_result
    if [[ "$ENABLE_COMMAND_VERIFY" == "true" ]]; then
        verify_result=$(verify_adb_command_defense "$command")
    else
        verify_result=$(verify_adb_command "$command")
    fi

    if [[ "$verify_result" -ne 0 ]]; then
        log_message "Command verification failed: $command"
        return
    fi
    local result=$(timeout "$TIMEOUT_ADB" adb -s "$DEVICE_SERIAL" shell "$sanitized_text" 2>/dev/null)
    local return_code=$?
    if [[ "$return_code" -ne 0 ]]; then
        log_message "ADB execution error: $command"
        if [[ "$ENABLE_ADAPTIVE_BEHAVIOR" == "true" ]]; then
          SLEEP_TIME=$((SLEEP_TIME*2)) #Adaptive sleep time.
          if [[ "$SLEEP_TIME" -gt 60 ]]; then
            SLEEP_TIME=60;
          fi
        fi
    else
        log_message "ADB command result: $result"
        if [[ "$ENABLE_ADAPTIVE_BEHAVIOR" == "true" ]]; then
          SLEEP_TIME="0.5" #Reset sleep time.
        fi
    fi
}

# Input Handling
generate_random_unicode_string() {
    local length=10
    local random_string=""
    for ((i=0; i<length; i++)); do
        local random_char=$((RANDOM % 65536))
        random_string+="$(printf '\\U%08x' "$random_char")"
    done
    echo "$random_string"
}

parse_input() {
  local input="$1"
  local parsed_input=""
  parsed_input=$(echo "$input" | sed 's/  */ /g' | tr '[:upper:]' '[:lower:]')
  echo "$parsed_input"
}

get_input() {
  if [[ "$INPUT_SOURCE" == "auto" ]]; then
    local input=$(generate_random_unicode_string)
    echo "$input"
  elif [[ "$INPUT_SOURCE" == "string" ]]; then
    read -p "Enter input string: " input
    echo "$input"
  else
    log_message "ERROR: Invalid INPUT_SOURCE. Must be 'auto' or 'string'."
    exit 1
  fi
}

# Main Loop
if [[ "$ENABLE_SYSTEM_CHECKS" == "true" ]] && [[ $(check_adb_binary) -ne 0 ]] ; then
  exit 1;
fi

if [[ "$ENABLE_SYSTEM_CHECKS" == "true" ]] && [[ $(check_device_connection) -ne 0 ]] ; then
  exit 1;
fi

while true; do
    local input=$(get_input)
    local parsed_input=$(parse_input "$input")
    send_adb_command "$parsed_input"
    sleep "$SLEEP_TIME"
done
