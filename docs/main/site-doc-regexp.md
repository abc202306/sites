

## vimscript-to-replace-generated-image-elems


```yaml
---
ctime: "2026-01-30T00:04:54+08:00"
mtime: "2026-01-30T00:04:54+08:00"
---
```


```vimscript
%s;\[(<img src=")(.*?)(".*?)(width="30">)\]\(.*?\);[$1https://raw.githubusercontent.com/abc202306/sites/refs/heads/main/$2$3width="50">](https://github.com/abc202306/sites/blob/main/$2);g
```
