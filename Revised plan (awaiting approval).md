Revised plan (awaiting approval)



&#x20;   Central data repo — apollo\_data/ + data\_repo module: sync CLI from all 3 sources in priority order (eod2 → Sheets → yfinance), Parquet canonical storage, manifest, --update merges new bars without re-downloading.- Yes go ahead.



&#x20;   Dynamic watchlists everywhere — schema-tolerant loader (sym / scripCode→.NS / list-of-strings), auto-discover all \*.json in the Apollo project folder for backtest app, live dashboard, NSE engine, chartink. Remove TEST\_STOCKS + DEFAULT\_SYMBOL; sort alphabetically; every stock with a CSV is testable. - Can we create one Json file combining all json files in project folder which is readable by Apollo (Keeping certain features intact such as if a stock belongs to IPO - the lists in All Apollo engines say so in the column?





&#x20;   Remove entry gates G1–G4 (gates.py, fundamentals.py, gate\_frame, GATE-BLOCKED events, ALERT\_GATE\_BLOCKED, PE wiring) — score-only entries.

&#x20;   Renko decision — need your call: (a) wire Pool R + hard gate so all 27 pts fire (score max 188, gate at R<5), or (b) delete the dead Renko code entirely. My lean: wire it, since you asked for "all scoring happening" — but it changes score distribution, so I want your confirmation. - Renko is to be wired - it will contribute to scoring. As of now it will not be a gate. But what details we can extract from Renko that shows us information about the trend that can be presented in a columnar format and adds value? Your suggestion



&#x20;   Robust eod2 parsing — flexible date formats + column subsetting so your real path works in the UI.- Yes go ahead.



&#x20;   Bat fixes — uncomment EOD2\_DIR → your path, write into central repo (--save-all), point --data-dir there. - Yes go ahead.

&#x20;   Verify — preflight additions: all 34 (+R if wired) + 12 IPO signals fire, watchlist loads from every JSON, repo roundtrip. Bump to v4.8, rebuild zip, deliver via HTTP preview.



Which way do you want the Renko pool — wire it (a) or remove it (b)? Once you confirm, I'll start. - Renko is to be wired - it will contribute to scoring. As of now it will not be a gate. But what details we can extract from Renko that shows us information about the trend that can be presented in a columnar format and adds value? Your suggestion.

After your response, we can go ahead with implementation.

(Make sure to update Project instructions md file with all the details that should be included, including details that are to be followed in subsequent iterations, instructions that are to be followed throughout project lifecycle.) - Update project MD

All project files should be included in a project zip file with proper versioning, which would be shared with the user.





