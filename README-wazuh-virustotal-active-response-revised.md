# Home SOC Lab – Part 2: Wazuh VirusTotal Integration and Active Response

## Project Overview

This project builds on Part 1 of the home SOC lab by adding VirusTotal enrichment and Wazuh Active Response.

The aim was to monitor a Windows directory with Wazuh File Integrity Monitoring (FIM), send detected file hashes to VirusTotal, and run a Windows response executable when VirusTotal reported a malicious file.

The workflow tested in this part was:

```text
File created on Windows
        ↓
Wazuh FIM detects the change
        ↓
Wazuh submits the file hash to VirusTotal
        ↓
VirusTotal identifies the EICAR test file
        ↓
Wazuh triggers remove-threat.exe
        ↓
The response checks whether the file still exists
```

During the final test, Microsoft Defender removed the EICAR file before the Wazuh response could delete it. Wazuh still detected the file, received the VirusTotal result, and launched the configured response.

## Objectives

- Reuse the Windows FIM directory from Part 1
- Manage FIM settings through a Wazuh agent group
- Integrate Wazuh with the VirusTotal API
- Test malware detection safely with the EICAR test file
- Confirm VirusTotal alerts in Wazuh Threat Hunting
- Build a Windows Active Response executable
- Trigger Active Response from a VirusTotal detection
- Investigate why Wazuh did not complete the file deletion
- Document the interaction between Wazuh and Microsoft Defender

## Project Status

### Completed

- [x] Reused the monitored Windows directory from Part 1
- [x] Configured Windows FIM through the Wazuh default agent group
- [x] Added the VirusTotal integration to the Wazuh manager
- [x] Verified FIM events for file creation, modification, and deletion
- [x] Used the EICAR test file for safe malware-detection testing
- [x] Confirmed VirusTotal detected the EICAR hash
- [x] Created `remove-threat.py`
- [x] Compiled the script into `remove-threat.exe`
- [x] Installed the executable in the Wazuh Active Response directory
- [x] Configured Wazuh to trigger the executable from rule `87105`
- [x] Confirmed the Active Response executable was launched
- [x] Investigated the remediation race between Wazuh and Microsoft Defender

## Detection and Response Flow

```text
C:\SOC\integrity-check
        │
        ├── Wazuh FIM monitors the directory
        │
        ├── Rule 554 records a new file
        │
        ├── VirusTotal checks the file hash
        │
        ├── Rule 87105 reports malicious detections
        │
        ├── Wazuh launches remove-threat.exe
        │
        └── Microsoft Defender may remove the file first
```

---

## 1. Reusing the FIM Directory from Part 1

The existing monitored directory from the first project was reused:

```text
C:\SOC\integrity-check
```

This directory had already been tested with harmless files and was producing Wazuh FIM events for file creation, modification, and deletion.

The expected FIM rules were:

| Rule ID | Description |
|---|---|
| `554` | File added |
| `550` | File modified |
| `553` | File deleted |

## 2. Configuring FIM Through the Agent Group

Instead of relying only on the local Windows `ossec.conf`, the configuration was managed centrally through the Wazuh dashboard:

```text
Agents Management → Groups → default → agent.conf
```

The group configuration targeted Windows agents and monitored the existing lab directory:

```xml
<agent_config os="Windows">
  <syscheck>
    <disabled>no</disabled>
    <directories realtime="yes"
                 check_all="yes"
                 report_changes="yes">C:\SOC\integrity-check</directories>
  </syscheck>
</agent_config>
```

This moved the FIM setting into Wazuh's centralized agent configuration instead of keeping it only on the endpoint.

![Windows agent group FIM configuration](screenshots/placeholder-windows-agent-group-fim-config.png)

*Replace `placeholder-windows-agent-group-fim-config.png` with the screenshot of the Wazuh group configuration.*

## 3. Adding the VirusTotal Integration

A VirusTotal API key was created and added to the Wazuh manager configuration:

```text
/var/ossec/etc/ossec.conf
```

The integration block followed the Wazuh VirusTotal configuration format:

```xml
<ossec_config>
  <integration>
    <name>virustotal</name>
    <api_key>REDACTED</api_key>
    <group>syscheck</group>
    <alert_format>json</alert_format>
  </integration>
</ossec_config>
```

The real API key is not included in the repository. Configuration examples use a placeholder such as `REDACTED`.

After editing the configuration, it was validated and the Wazuh manager was restarted.

```bash
sudo bash -c 'cd /var/ossec && ./bin/wazuh-analysisd -t'
sudo systemctl restart wazuh-manager
sudo systemctl status wazuh-manager --no-pager
```

![VirusTotal integration configuration](screenshots/placeholder-virustotal-integration-config.png)

*Replace `placeholder-virustotal-integration-config.png` with a redacted screenshot of the VirusTotal integration block.*

## 4. Using the EICAR Test File

EICAR is a harmless, industry-standard test file used to check antivirus and endpoint detection workflows. It contains no real malicious code, but security products are designed to detect it as a test threat.

The file was retrieved from the official EICAR site with PowerShell:

```powershell
Invoke-WebRequest `
  -Uri "https://secure.eicar.org/eicar.com.txt" `
  -OutFile "C:\SOC\integrity-check\eicar.txt"
```

The test file was not executed. Placing it in the monitored directory was enough to trigger FIM and antivirus inspection.

![EICAR file in monitored directory](screenshots/placeholder-eicar-file-in-directory.png)

*Replace `placeholder-eicar-file-in-directory.png` with a screenshot showing the EICAR file in the monitored directory.*

## 5. Confirming the VirusTotal Alert

After the EICAR file appeared in the monitored directory, Wazuh generated the FIM event and submitted its hash to VirusTotal.

In Wazuh Threat Hunting, the VirusTotal result showed that multiple antivirus engines identified the file.

The most important event in this stage was:

```text
Rule ID: 87105
```

This confirmed the following path was working:

```text
Windows FIM
→ Wazuh manager
→ VirusTotal API
→ Wazuh Threat Hunting alert
```

![VirusTotal malicious-file alert](screenshots/placeholder-virustotal-eicar-alert.png)

*Replace `placeholder-virustotal-eicar-alert.png` with the screenshot showing the VirusTotal detection in Threat Hunting.*

## 6. Creating the Active Response Script

A Python response script was created as:

```text
remove-threat.py
```

The script reads the Active Response JSON input, extracts the file path from the VirusTotal alert, and tries to delete the detected file.

The source script is stored in:

```text
scripts/remove-threat.py
```

The Windows build folder was:

```text
C:\SOC\ActiveResponse
```

![remove-threat.py script](screenshots/placeholder-remove-threat-python-script.png)

*Replace `placeholder-remove-threat-python-script.png` with a screenshot of the Python script.*

## 7. Compiling the Script into an Executable

PyInstaller was used to compile the Python script into a standalone Windows executable:

```powershell
Set-Location "C:\SOC\ActiveResponse"

py -m PyInstaller `
  --onefile `
  --clean `
  --name remove-threat `
  ".\remove-threat.py"
```

The resulting executable was created at:

```text
C:\SOC\ActiveResponse\dist\remove-threat.exe
```

It was then copied into the Wazuh Windows agent Active Response directory:

```powershell
Copy-Item `
  "C:\SOC\ActiveResponse\dist\remove-threat.exe" `
  "C:\Program Files (x86)\ossec-agent\active-response\bin\remove-threat.exe" `
  -Force
```

![Compiled remove-threat executable](screenshots/placeholder-remove-threat-executable.png)

*Replace `placeholder-remove-threat-executable.png` with a screenshot showing the compiled executable.*

## 8. Configuring Wazuh Active Response

The Wazuh manager was configured to run `remove-threat.exe` when VirusTotal rule `87105` fired.

The following command and Active Response configuration were added to the Wazuh manager:

```xml
<ossec_config>
  <command>
    <name>remove-threat</name>
    <executable>remove-threat.exe</executable>
    <timeout_allowed>no</timeout_allowed>
  </command>

  <active-response>
    <disabled>no</disabled>
    <command>remove-threat</command>
    <location>local</location>
    <rules_id>87105</rules_id>
  </active-response>
</ossec_config>
```

The manager configuration was validated before restarting the service.

```bash
sudo bash -c 'cd /var/ossec && ./bin/wazuh-analysisd -t'
sudo systemctl restart wazuh-manager
```

## 9. Observed File Deletion

The EICAR file appeared briefly in the monitored directory and then disappeared.

Wazuh FIM recorded the deletion:

```text
Rule ID: 553
Action: deleted
```

![FIM file deletion event](screenshots/placeholder-fim-eicar-deleted.png)

*Replace `placeholder-fim-eicar-deleted.png` with the screenshot showing the file deletion in Wazuh FIM.*

The deletion event alone did not show which product removed the file, so the Windows Active Response log was checked.

## 10. Investigating the Active Response Result

The Windows Active Response log was reviewed at:

```text
C:\Program Files (x86)\ossec-agent\active-response\active-responses.log
```

The log confirmed that Wazuh launched `remove-threat.exe`, but the executable reported:

```text
Error removing threat: File does not exist
```

From the log, the following was confirmed:

1. Wazuh received the VirusTotal alert.
2. Rule `87105` triggered the configured response.
3. `remove-threat.exe` ran on the Windows endpoint.
4. The file was already gone when the executable attempted deletion.

![Wazuh Active Response file-not-found result](screenshots/placeholder-active-response-file-not-found.png)

*Replace `placeholder-active-response-file-not-found.png` with the screenshot showing the Active Response log.*

## 11. Final Outcome

The final observed sequence was:

```text
EICAR file created
        ↓
Wazuh FIM detects the file
        ↓
VirusTotal identifies the hash
        ↓
Wazuh triggers remove-threat.exe
        ↓
Microsoft Defender removes the file first
        ↓
remove-threat.exe reports that the file no longer exists
        ↓
Wazuh FIM records the deletion
```

Microsoft Defender consistently removed the EICAR file before Wazuh could complete its own deletion attempt, including during tests where Defender protection settings had been reduced in the isolated VM.

The Wazuh integration itself worked: FIM detected the file, VirusTotal returned a malicious result, and Active Response launched. Defender simply completed remediation first.

### Final conclusion

> Wazuh detected the EICAR test file through FIM, sent the hash to VirusTotal, and triggered the configured Windows Active Response executable. Microsoft Defender removed the file first, and the response log showed that `remove-threat.exe` ran after the target was already gone.

This result highlights why response logs matter: the deletion event showed that the file disappeared, while the Active Response log identified that Wazuh did not perform the removal.

## 12. Evidence Summary

| Stage | Evidence |
|---|---|
| FIM monitoring enabled | Windows agent group configuration |
| File added | Wazuh rule `554` |
| VirusTotal malicious result | Wazuh rule `87105` |
| Active Response triggered | `active-responses.log` |
| File deleted | Wazuh rule `553` |
| Wazuh did not perform deletion | `File does not exist` response |
| Competing endpoint remediation | Microsoft Defender detection history |

## 13. Skills Demonstrated

- Wazuh centralized agent configuration
- File Integrity Monitoring
- External API integration
- VirusTotal threat-intelligence enrichment
- Windows endpoint monitoring
- Python scripting
- PyInstaller executable creation
- Wazuh Active Response configuration
- XML configuration and validation
- Threat Hunting
- Log analysis
- Root-cause investigation
- Layered security analysis
- Technical documentation

## 14. Repository Structure

```text
project-root/
├── README.md
├── configs/
│   ├── agent.conf.example
│   ├── virustotal-integration.xml
│   └── active-response.xml
├── scripts/
│   └── remove-threat.py
└── screenshots/
    ├── 01-windows-agent-group-fim-config.png
    ├── 02-virustotal-integration-config.png
    ├── 03-eicar-file-in-directory.png
    ├── 04-virustotal-eicar-alert.png
    ├── 05-remove-threat-python-script.png
    ├── 06-remove-threat-executable.png
    ├── 07-fim-eicar-deleted.png
    └── 08-active-response-file-not-found.png
```

Do not upload:

- VirusTotal API keys
- Wazuh administrator credentials
- Windows passwords
- VMware encryption passwords
- Unnecessary private IP addresses
- Unredacted screenshots containing secrets

## 15. Future Improvements

- Test the same workflow on a Linux endpoint without competing antivirus remediation
- Add a custom Wazuh rule for “threat already absent”
- Create a dashboard visualization for the full detection chain
- Record Defender and Wazuh timestamps to compare response speed
- Send the VirusTotal alert to an incident-response platform
- Add email or webhook notification
- Test a benign Active Response action separately from antivirus removal
- Integrate TheHive or Shuffle for case creation and automation

## References

- [Wazuh: Detecting and removing malware using VirusTotal integration](https://documentation.wazuh.com/current/proof-of-concept-guide/detect-remove-malware-virustotal.html)
- [Wazuh: VirusTotal integration](https://documentation.wazuh.com/current/user-manual/capabilities/malware-detection/virus-total-integration.html)
- [EICAR Anti-Malware Test File](https://www.eicar.org/download-anti-malware-testfile/)
- [Microsoft Defender: Validate antimalware protection with EICAR](https://learn.microsoft.com/en-us/defender-endpoint/validate-antimalware)

## Disclaimer

This project was performed in an isolated virtual lab for educational purposes.

The EICAR test file is not real malware. It is an industry-standard test file designed to trigger antivirus products safely.

Do not test security controls on systems that you do not own or have explicit permission to use.
