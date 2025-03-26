# plesk_emailcounter_extension
This extension for Plesk provides the admin with an overview of the total amounts of succesfully sent e-mails per day for the last 30 days.

**Step 1.**
Create the extension:
`plesk bin extension --create emailcounter`

and register it: `plesk bin extension --register emailcounter`

**Step 2.**
Edit the /usr/local/psa/admin/plib/modules/emailcounter/views/scripts/index/index.phtml by inserting this code:
```
<?php
// Copyright 2024.

$jsonFile = '/usr/local/psa/var/logs/mail_sent_history.json';
$today = date("Y-m-d");

// Get live email count from the helper script
function getLiveEmailCount() {
    $output = shell_exec("sudo /usr/local/bin/get_live_mail_count.sh");
    return is_numeric(trim($output)) ? trim($output) : "-";
}

// Load historical data excluding today
$filteredTotals = [];
if (file_exists($jsonFile)) {
    $jsonData = json_decode(file_get_contents($jsonFile), true);
    if (is_array($jsonData)) {
        foreach ($jsonData as $entry) {
            if ($entry['date'] !== $today) {
                $filteredTotals[] = $entry;
            }
        }
    }
}
usort($filteredTotals, fn($a, $b) => strcmp($b['date'], $a['date']));

$liveEmailCount = getLiveEmailCount();
?>

<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Plesk Extension Output</title>
    <style>
        body { font-family: Arial, sans-serif; padding: 20px; }
        pre { background: #f4f4f4; padding: 10px; border-radius: 5px; }
        th, td { padding-right: 15px; }
    </style>
</head>
<body>
    <h2>Dit is het aantal succesvol verzonden e-mails op <?= htmlspecialchars($today) ?> tussen 00:00 en nu:</h2>

    <p><strong>
    <pre><?php echo htmlspecialchars($liveEmailCount); ?></pre>
    </strong></p>

    <h3>Overzicht van de afgelopen 30 dagen:</h3>
    <table>
        <tr>
            <th>Datum</th>
            <th>Totaal verzonden e-mails</th>
        </tr>
        <?php foreach ($filteredTotals as $entry): ?>
            <tr>
                <td><?php echo htmlspecialchars($entry['date']); ?></td>
                <td><?php echo htmlspecialchars($entry['total'] ?? "-"); ?></td>
            </tr>
        <?php endforeach; ?>
    </table>
</body>
</html>
```

**Step 3.**
Create a bashscript that pulls the total amounts of succesfully sent e-mail messages from the maillog and logs it in a json file that's accessible to the psaadm user: `/usr/local/bin/mail_log_processor.sh`

Enter this code inside the file:
```
#!/bin/bash

LOGFILE="/usr/local/psa/var/logs/mail_sent_history.json"

LAST_DAY_LOG=$(date -d "yesterday" "+%b %d")  # For grepping logs (e.g., "Mar 23")
LAST_DAY_JSON=$(date -d "yesterday" "+%Y-%m-%d")  # For storing in JSON (e.g., "2025-03-23")

# Grep for sent messages on the current day, including gzipped maillogs and store them in a variable
TOTAL=$(zgrep -i "$LAST_DAY_LOG" /var/log/maillog* | grep "status=sent (250" | wc -l)

# Ensure JSON file exists
if [ ! -f "$LOGFILE" ]; then
    echo "[]" > "$LOGFILE"
fi

# Read the existing JSON file
JSON=$(cat "$LOGFILE")

# Ensure the JSON is an array
if ! echo "$JSON" | jq -e "type == \"array\"" > /dev/null; then
    JSON="[]"
fi

# Remove existing entry for today if it exists (to prevent duplicates)
JSON=$(echo "$JSON" | jq --arg date "$LAST_DAY_JSON" 'map(select(.date != $date))')

# Append the new entry
JSON=$(echo "$JSON" | jq --arg date "$LAST_DAY_JSON" --arg total "$TOTAL" \
    '. += [{"date": $date, "total": ($total | tonumber)}] | sort_by(.date) | reverse | .[:30]')

# Save the updated JSON back to the file
echo "$JSON" > "$LOGFILE"
```
and make it executable: `chmod +x /usr/local/bin/mail_log_processor.sh`

**Step 4.**
Create a systemd service and timer file: `/etc/systemd/system/mail_log_processor.service` and `/etc/systemd/system/mail_log_processor.timer`

Insert this code inside the mail_log_processor.service:
```
[Unit]
Description=Daily Succefully Sent E-mail Counter

[Service]
Type=oneshot
ExecStart=/usr/local/bin/mail_log_processor.sh
```

and this inside the mail_log_processor.timer:
```
[Unit]
Description=Daily Succefully Sent E-mail Counter

[Timer]
OnCalendar=*-*-* 00:00:00
Persistent=true

[Install]
WantedBy=timers.target
```

then reload systemd and enable the timer by executing these commands:
```
systemctl daemon-reload
systemctl enable mail_log_processor.timer
systemctl start mail_log_processor.timer
```

**Step 5.**
Create a helper script for the live e-mailcount `/usr/local/bin/get_live_mail_count.sh` and insert this code in it:
```
#!/bin/bash

TODAY_DATE=$(date "+%b %d")

# Count sent emails from today's logs
TOTAL=$(zgrep -i "$TODAY_DATE" /var/log/maillog* | grep "status=sent (250" | wc -l)

echo $TOTAL
```
then make it executable:
`chmod +x /usr/local/bin/get_live_mail_count.sh`

**Step 6.**
Edit the approproiate META info inside `/usr/local/psa/admin/plib/modules/emailcounter/meta.xml`

Step 7. (extremely important...! ;))
Add the complementary icons for the extension inside `/usr/local/psa/admin/share/modules/emailcounter/_meta/icons`
(note that you need to keep their names as they are: 32x32.png, 64x64.png, and 128x128.png)
