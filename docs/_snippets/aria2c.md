---
---
## Objective
Download and resume with [`aria2c`](https://github.com/aria2/aria2)

## Download

```powershell
aria2c `
  --continue=true `
  --auto-file-renaming=false `
  --allow-overwrite=false `
  --input-file=urls.txt `
  --save-session=unfinished.txt `
  --save-session-interval=30
```

## Resume

```powershell
aria2c `
  --continue=true `
  --auto-file-renaming=false `
  --allow-overwrite=false `
  --input-file=unfinished.txt `
  --save-session=unfinished.txt
```
