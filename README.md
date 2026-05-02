# CalibreKindleAutoImport
I have made this automated system for calibre, it works by scanning for new files added to the calibre library and it sends them by mail to the kindle mail and by that it auto downloads on the kindle.
The script knows to convert CBR/CBZ/PDF → EPUB for the best handeling on the kindle, currently it works the best in english.

1. Downloading the required components for it to work(It includes calibre, if you already have it installed you can remove it from the commend)

```bash
sudo apt update && sudo apt install -y wget inotify-tools msmtp msmtp-mta mailutils poppler-utils unrar zip
sudo -v && wget -nv -O- https://download.calibre-ebook.com/linux-installer.sh | sudo sh /dev/stdin
```
If you have problem while downloading try to change the DNS setting by running this commend. 

```bash
sudo sed -i 's|il.archive.ubuntu.com|archive.ubuntu.com|g' /etc/apt/sources.list
``` 
2. Create 2 folders, one for the kindle script and one for the calibre library(Use the same file name, if not you would need to edit that in the script).
```bash
mkdir -p /home/USER/calibre-library
mkdir -p /home/USER/kindle-sync
```
3. Initialize Calibre library
```bash
calibredb list --with-library /home/USER/calibre-library
```
4. Create Calibre user(if you having problems with this step go to https://manual.calibre-ebook.com/generated/en/calibre-server.html).
```bash
calibre-server --userdb /home/asaf/calibre-users.sqlite --manage-users
```
5. Create Calibre service
```bash
sudo nano /etc/systemd/system/calibre-server.service
```
in the calibre-server.service paste:
```bash
[Unit]
Description=Calibre Content Server
After=network.target
#replace USER with your created user, and linux user.
[Service]
User=USER
ExecStart=/usr/bin/calibre-server /home/USER/calibre-library --port 8081 --enable-local-write --enable-auth --userdb /home/USER/calibre-users.sqlite
Restart=always

[Install]
WantedBy=multi-user.target
```
after that start the server.
```bash
sudo systemctl daemon-reload
sudo systemctl enable --now calibre-server
```
ahead to http://SERVER-IP:8081 to see that everything works correctly.
6. Configure Gmail sender
```bash
nano /home/USER/.msmtprc
```
paste:
```bash
defaults
auth           on
tls            on
tls_trust_file /etc/ssl/certs/ca-certificates.crt
logfile        /home/USER/.msmtp.log #replace user with your linux user.

account        gmail
host           smtp.gmail.com
port           587
from           YOUR_GMAIL@gmail.com #replace with your gmail account, I recommend to open a new one for this.
user           YOUR_GMAIL@gmail.com
# replace with your google app password, you can get it from here: myaccount.google.com/apppasswords(You must have 2FA enabled!)
password       YOUR_GOOGLE_APP_PASSWORD 


account default : gmail
```
make it executable:
```bash
chmod 600 /home/asaf/.msmtprc
```
ahead to amazon "Manage your content and devices" and ahead to perfrences.
1. ahead all the way down and add your mail to the "Approved Personal Document E-mail List" (The mail you used for the script).
2. copy your kindle email.

7. Create Kindle sync script
```bash
nano /home/asaf/auto-send-from-calibre.sh
```
then paste(and change the kindle email and path to the calibre library: 
```bash
#!/bin/bash

LIBRARY="/home/USER/calibre-library"
KINDLE_EMAIL="YOUR_KINDLE_EMAIL"

BASE_DIR="/home/USER/kindle-sync"
LOG_FILE="$BASE_DIR/kindle-sync.log"
SENT_LOG="$BASE_DIR/sent.log"
FAILED_DIR="$BASE_DIR/failed"
TMP_DIR="$BASE_DIR/tmp"

MAX_MB=24
MAX_BYTES=$((MAX_MB * 1024 * 1024))
RETRIES=3

mkdir -p "$BASE_DIR" "$FAILED_DIR" "$TMP_DIR"
touch "$LOG_FILE" "$SENT_LOG"

log() {
    echo "$(date '+%Y-%m-%d %H:%M:%S') | $1" | tee -a "$LOG_FILE"
}

send_to_kindle() {
    local FILE_PATH="$1"
    local SUBJECT="$2"

    local SIZE
    SIZE=$(stat -c%s "$FILE_PATH")

    if [ "$SIZE" -gt "$MAX_BYTES" ]; then
        log "SKIPPED: File too large over ${MAX_MB}MB: $FILE_PATH"
        cp "$FILE_PATH" "$FAILED_DIR/"
        return 1
    fi

    for TRY in $(seq 1 "$RETRIES"); do
        log "Sending attempt $TRY/$RETRIES: $FILE_PATH"

        echo "Book attached" | mail -s "$SUBJECT" -A "$FILE_PATH" "$KINDLE_EMAIL"

        if [ $? -eq 0 ]; then
            log "SENT: $FILE_PATH"
            return 0
        fi

        sleep 10
    done

    log "FAILED after retries: $FILE_PATH"
    cp "$FILE_PATH" "$FAILED_DIR/"
    return 1
}

process_book() {
    local FULL_PATH="$1"

    if [ ! -f "$FULL_PATH" ]; then
        return
    fi

    if grep -Fxq "$FULL_PATH" "$SENT_LOG"; then
        log "Already processed, skipping: $FULL_PATH"
        return
    fi

    local FILE
    FILE="$(basename "$FULL_PATH")"

    local EXT
    EXT="${FILE##*.}"
    EXT="$(echo "$EXT" | tr '[:upper:]' '[:lower:]')"

    local BASENAME
    BASENAME="${FILE%.*}"

    local WORK_DIR="$TMP_DIR/${BASENAME}_work"
    local OUTPUT_FILE="$TMP_DIR/${BASENAME}.epub"
    local INPUT_FILE="$FULL_PATH"

    rm -rf "$WORK_DIR" "$OUTPUT_FILE"
    mkdir -p "$WORK_DIR"

    log "Detected: $FULL_PATH"

    case "$EXT" in
        epub)
            log "EPUB detected, sending directly."
            if send_to_kindle "$FULL_PATH" "$BASENAME"; then
                echo "$FULL_PATH" >> "$SENT_LOG"
            fi
            ;;

        pdf)
            log "PDF detected, converting to EPUB."
            ebook-convert "$FULL_PATH" "$OUTPUT_FILE"

            if [ -f "$OUTPUT_FILE" ]; then
                if send_to_kindle "$OUTPUT_FILE" "convert"; then
                    echo "$FULL_PATH" >> "$SENT_LOG"
                fi
            else
                log "Conversion failed: $FULL_PATH"
                cp "$FULL_PATH" "$FAILED_DIR/"
            fi
            ;;

        cbz)
            log "CBZ detected, converting to EPUB."
            ebook-convert "$FULL_PATH" "$OUTPUT_FILE"

            if [ -f "$OUTPUT_FILE" ]; then
                if send_to_kindle "$OUTPUT_FILE" "convert"; then
                    echo "$FULL_PATH" >> "$SENT_LOG"
                fi
            else
                log "Conversion failed: $FULL_PATH"
                cp "$FULL_PATH" "$FAILED_DIR/"
            fi
            ;;

        cbr)
            log "CBR detected, extracting and converting to CBZ first."

            unrar x -o+ "$FULL_PATH" "$WORK_DIR/"

            if [ $? -ne 0 ]; then
                log "CBR extraction failed: $FULL_PATH"
                cp "$FULL_PATH" "$FAILED_DIR/"
                return
            fi

            local CBZ_FILE="$TMP_DIR/${BASENAME}.cbz"

            (
                cd "$WORK_DIR" || exit 1
                zip -r "$CBZ_FILE" ./*
            )

            if [ ! -f "$CBZ_FILE" ]; then
                log "CBZ creation failed: $FULL_PATH"
                cp "$FULL_PATH" "$FAILED_DIR/"
                return
            fi

            ebook-convert "$CBZ_FILE" "$OUTPUT_FILE"

            if [ -f "$OUTPUT_FILE" ]; then
                if send_to_kindle "$OUTPUT_FILE" "convert"; then
                    echo "$FULL_PATH" >> "$SENT_LOG"
                fi
            else
                log "Conversion failed: $FULL_PATH"
                cp "$FULL_PATH" "$FAILED_DIR/"
            fi
            ;;

        *)
            log "Unsupported file type, skipping: $FULL_PATH"
            ;;
    esac

    rm -rf "$WORK_DIR" "$OUTPUT_FILE" "$TMP_DIR/${BASENAME}.cbz"
}

log "Kindle sync service started."

inotifywait -m -r -e close_write,moved_to --format "%w%f" "$LIBRARY" | while read FULL_PATH
do
    process_book "$FULL_PATH"
done
```
8. Create Kindle sync service
```bash
sudo nano /etc/systemd/system/kindle-sync.service
```
paste(and change):
```bash
[Unit]
Description=Auto Send Books to Kindle
After=network.target calibre-server.service

[Service]
User=USER
ExecStart=/home/USER/auto-send-from-calibre.sh
Restart=always

[Install]
WantedBy=multi-user.target
```
then paste:
```bash
sudo systemctl daemon-reload
sudo systemctl enable --now kindle-sync
```
9. Check status
```bash
systemctl status calibre-server
systemctl status kindle-sync
journalctl -u kindle-sync -f
```
now you can add a file to the calibre library and get it sent out(you can check in the logs that everything works).
The kindle will take around 1-2 minutes to recieve it correctly.

