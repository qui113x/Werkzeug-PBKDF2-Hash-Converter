## Werkzeug-PBKDF2-Hash-Converter
# A simple python script for converting Werkzeug PBKDF2 hashes to their corresponding Hachcat format and vice versa


!) This is a simple python script for converting pbkdf2_sha256 hashes into Hashcat compatible hashes. You can also work backwards from Hashcat compatible hashes into pbkdf2 hashes. 

2) This script also contains a batch function that allows for the conversion of multiple hashes from a single input file.

```
#!/usr/bin/env python3
import base64
import sys

# ----------------------------------------------------------
# Colors (ANSI escape codes – 256-color)
# ----------------------------------------------------------
RESET = "\033[0m"
BOLD = "\033[1m"

RED = "\033[31m"
CYAN = "\033[36m"

ELECTRIC_YELLOW = "\033[38;5;226m"
NEON_PURPLE = "\033[38;5;201m"
ELECTRIC_GREEN = "\033[38;5;82m"

def err(text: str) -> str:
    return RED + text + RESET

def info(text: str) -> str:
    return CYAN + text + RESET

def num(text: str) -> str:
    # for menu numbering
    return ELECTRIC_YELLOW + text + RESET

def wk2hc_color(text: str) -> str:
    # Werkzeug → Hashcat output
    return NEON_PURPLE + text + RESET

def hc2wk_color(text: str) -> str:
    # Hashcat → Werkzeug output
    return ELECTRIC_GREEN + text + RESET

def neutral_warn(text: str) -> str:
    return ELECTRIC_YELLOW + text + RESET

# Generic fallback (used only in CLI auto mode)
def auto_success_color(text: str, src_type: str) -> str:
    if src_type == "werkzeug":
        return wk2hc_color(text)
    elif src_type == "hashcat":
        return hc2wk_color(text)
    return text

# ----------------------------------------------------------
# Utility functions
# ----------------------------------------------------------

def ascii_to_b64(s: str) -> str:
    return base64.b64encode(s.encode()).decode()

def hex_to_b64(h: str) -> str:
    return base64.b64encode(bytes.fromhex(h)).decode()

def b64_to_ascii(b: str) -> str:
    return base64.b64decode(b).decode()

def b64_to_hex(b: str) -> str:
    return base64.b64decode(b).hex()

# ----------------------------------------------------------
# Werkzeug → Hashcat
# - Supports:
#   pbkdf2:sha256:<iter>$<salt>$<hex_digest>
#   sha256:<iter>$<salt>$<hex_digest>
#
# Output (Hashcat-style):
#   sha256:<iter>:<salt_b64>:<digest_b64>
# ----------------------------------------------------------

def werkzeug_to_hashcat(wk_hash: str) -> str:
    try:
        wk_hash = wk_hash.strip()

        # Recognize and normalize variations
        if wk_hash.startswith("pbkdf2:sha256:"):
            # pbkdf2:sha256:<iter>$<salt>$<hex>
            _, _, iter_and_rest = wk_hash.split(":", 2)
        elif wk_hash.startswith("sha256:") and "$" in wk_hash:
            # sha256:<iter>$<salt>$<hex>
            _, iter_and_rest = wk_hash.split(":", 1)
        else:
            raise ValueError("Not a recognized Werkzeug pbkdf2/sha256 format.")

        # iter_and_rest: "<iter>$<salt>$<hex_digest>"
        iterations, salt, hex_digest = iter_and_rest.split("$", 2)

        salt_b64 = ascii_to_b64(salt)
        digest_b64 = hex_to_b64(hex_digest)

        return f"sha256:{iterations}:{salt_b64}:{digest_b64}"

    except Exception as e:
        return f"[ERROR] Werkzeug parsing failed: {e}"

# ----------------------------------------------------------
# Hashcat → Werkzeug
# Input (Hashcat):
#   sha256:<iter>:<salt_b64>:<digest_b64>
#
# Output (Werkzeug-style):
#   pbkdf2:sha256:<iter>$<salt_ascii>$<hex_digest>
# ----------------------------------------------------------

def hashcat_to_werkzeug(hc_hash: str) -> str:
    try:
        hc_hash = hc_hash.strip()
        algo, iterations, salt_b64, digest_b64 = hc_hash.split(":")

        if algo != "sha256":
            raise ValueError("Not a Hashcat PBKDF2-SHA256 (sha256:...) format.")

        salt_ascii = b64_to_ascii(salt_b64)
        hex_digest = b64_to_hex(digest_b64)

        return f"pbkdf2:sha256:{iterations}${salt_ascii}${hex_digest}"

    except Exception as e:
        return f"[ERROR] Hashcat parsing failed: {e}"

# ----------------------------------------------------------
# Auto detection
# ----------------------------------------------------------

def detect_format(h: str) -> str:
    h = h.strip()

    # Werkzeug-style (either pbkdf2:sha256:... or sha256:... with $ pieces)
    if h.startswith("pbkdf2:sha256:"):
        return "werkzeug"
    if h.startswith("sha256:") and "$" in h:
        return "werkzeug"

    # Hashcat-style: sha256:<iter>:<b64>:<b64>
    if h.startswith("sha256:"):
        parts = h.split(":")
        if len(parts) == 4 and "$" not in h:
            return "hashcat"

    return "unknown"

# ----------------------------------------------------------
# Batch conversion for file input
# ----------------------------------------------------------

def convert_line(line: str) -> str:
    h = line.strip()
    if not h:
        return ""
    t = detect_format(h)
    if t == "werkzeug":
        return werkzeug_to_hashcat(h)
    elif t == "hashcat":
        return hashcat_to_werkzeug(h)
    else:
        return f"[UNKNOWN FORMAT] {h}"

# ----------------------------------------------------------
# Interactive Menu
# ----------------------------------------------------------

def interactive():
    print()
    print(BOLD + CYAN + "=== PBKDF2 SHA256 Hash Converter ===" + RESET)
    print(num("1)") + " Werkzeug → Hashcat")
    print(num("2)") + " Hashcat → Werkzeug")
    print(num("3)") + " Convert hashes from file (auto-detect each line)")
    print(num("4)") + " Exit\n")

    choice = input(info("Choose an option (1–4): ") + " ").strip()

    if choice == "1":
        print()
        wk = input(info("Enter Werkzeug hash:\n") + "> ").strip()
        print()
        print(info("Converted (Werkzeug → Hashcat):"))
        result = werkzeug_to_hashcat(wk)
        if result.startswith("[ERROR]"):
            print(err(result))
        else:
            print(wk2hc_color(result))

    elif choice == "2":
        print()
        hc = input(info("Enter Hashcat hash:\n") + "> ").strip()
        print()
        print(info("Converted (Hashcat → Werkzeug):"))
        result = hashcat_to_werkzeug(hc)
        if result.startswith("[ERROR]"):
            print(err(result))
        else:
            print(hc2wk_color(result))

    elif choice == "3":
        print()
        path = input(info("Enter filename: ") + "").strip()
        try:
            with open(path, "r") as f:
                lines = f.read().splitlines()
            print()
            print(info("Converted hashes:\n"))
            for ln in lines:
                if not ln.strip():
                    continue
                src_type = detect_format(ln)
                result = convert_line(ln)
                if result.startswith("[ERROR]"):
                    print(err(result))
                elif result.startswith("[UNKNOWN FORMAT]"):
                    print(neutral_warn(result))
                else:
                    print(auto_success_color(result, src_type))
        except Exception as e:
            print(err(f"[ERROR] Could not read file: {e}"))

    elif choice == "4":
        print(info("Exiting."))
        sys.exit(0)
    else:
        print(err("Invalid choice."))

    print()
    interactive()

# ----------------------------------------------------------
# Main
# ----------------------------------------------------------

if __name__ == "__main__":
    # Direct single-hash mode: python3 hashconv.py "<hash>"
    if len(sys.argv) == 2:
        line = sys.argv[1]
        src_type = detect_format(line)
        result = convert_line(line)
        if result.startswith("[ERROR]"):
            print(err(result))
        elif result.startswith("[UNKNOWN FORMAT]"):
            print(neutral_warn(result))
        else:
            print(auto_success_color(result, src_type))
        sys.exit(0)

    # Otherwise, go interactive
    interactive()
```
