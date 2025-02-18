---
title: Windows.Applications.MegaSync
hidden: true
tags: [Client Artifact]
---

This artifact will parse MEGASync logs and enables using regex to search for
entries of interest.

With UploadLogs selected a copy of the logs are uploaded to the server.
SearchVSS enables search over VSS and dedup support.


```yaml
name: Windows.Applications.MegaSync
description: |
  This artifact will parse MEGASync logs and enables using regex to search for
  entries of interest.

  With UploadLogs selected a copy of the logs are uploaded to the server.
  SearchVSS enables search over VSS and dedup support.

author: "Matt Green - @mgreen27"

reference:
  - https://attack.mitre.org/techniques/T1567/002/

precondition: SELECT OS From info() where OS = 'windows'

parameters:
  - name: LogFiles
    default: 'C:\Users\*\AppData\Local\Mega Limited\MEGAsync\logs\*.log'
  - name: SearchRegex
    description: "Regex of strings to search in line."
    default: 'Transfer\s\(UPLOAD\)|upload\squeue|local\sfile\saddition\sdetected|Sync\s-\ssending\sfile|\"user\"'
    type: regex
  - name: WhitelistRegex
    description: "Regex of strings to leave out of output."
    default:
    type: regex
  - name: SearchVSS
    description: "Add VSS into query."
    type: bool
  - name: UploadLogs
    description: "Upload MEGASync logs."
    type: bool


sources:
  - query: |
      -- Find target files
      LET files = SELECT *
        FROM if(condition=SearchVSS,
            then= {
                SELECT *
                FROM Artifact.Windows.Search.VSS(SearchFilesGlob=LogFiles)
            },
            else= {
                SELECT *, FullPath as Source
                FROM glob(globs=LogFiles)
            })

      -- Collect all Lines in scope of regex search
      LET output <= SELECT * FROM foreach(row=files,
          query={
              SELECT Line, FullPath,
                Mtime,
                Atime,
                Ctime,
                Size
              FROM parse_lines(filename=FullPath,accessor='file')
              WHERE TRUE
                AND Line =~ SearchRegex
                AND NOT if(condition= WhitelistRegex,
                    then= Line=~WhitelistRegex,
                    else = false)
          })
        GROUP BY Line

      SELECT
        Line as RawLine,
        FullPath
      FROM output


  - name: LogFiles
    query: |
        SELECT
            FullPath,
            if(condition=UploadLogs,
                then= upload(file=FullPath,accessor='ntfs')
                ) as Upload,
            'MEGAsync logfile' as Description,
            Mtime,
            Atime,
            Ctime,
            Size
        FROM output
        GROUP BY FullPath

```
