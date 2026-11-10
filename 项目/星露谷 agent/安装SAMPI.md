
# 安装SAMPI

版本号：4.5.2

备份游戏

```bash
mkdir -p "$HOME/Games/backups"

BACKUP="$HOME/Games/backups/Stardew Valley.app"

if [ -d "$BACKUP" ]; then
    echo "Backup already exists."
else
    ditto \
      "/Applications/Stardew Valley.app" \
      "$BACKUP"
    echo "Backup completed."
fi
```

