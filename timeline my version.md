# Investigation Timeline
| Sequence | Activity | Observation / Result |
|---|---|---|
| 1 | Checked `local_decoder.xml` | Existing example decoder found |
| 2 | Created custom application log directory | `/var/log/custom-app` created |
| 3 | Created authentication failure test log | Controlled event written |
| 4 | Developed custom decoder | `custom-auth` configured |
| 5 | Tested timestamp-prefixed log | Built-in `windows-date-format` decoder matched instead |
| 6 | Corrected decoder field mapping | Used `user,srcip,extra_data` |
| 7 | Tested decoder without timestamp | Custom decoder matched successfully |
| 8 | Developed custom detection rule | Rule ID `100200`, level 5 |
| 9 | Tested decoder and rule | Phase 2 and Phase 3 successful |
| 10 | Configured custom log monitoring | Added `localfile` entry to `ossec.conf` |
| 11 | Restarted Wazuh Manager | Service reported active |
| 12 | Generated fresh authentication event | Event appended to custom log |
| 13 | Verified Logcollector | Custom log shown as being analyzed |
| 14 | Checked `alerts.json` | Custom alert found |
| 15 | Reviewed alert fields | Expected username, IP and failure reason confirmed |

