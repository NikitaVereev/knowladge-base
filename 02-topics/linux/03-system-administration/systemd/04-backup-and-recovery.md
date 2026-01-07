# Backup and Recovery

## NAME

Резервное копирование и восстановление: стратегии, инструменты и тестирование.

## SYNOPSIS

```bash
# Резервное копирование
sudo rsync -av /home/ /mnt/backup/home/
sudo tar -czf backup.tar.gz /home/
sudo dd if=/dev/sda of=backup.img

# Восстановление
sudo rsync -av /mnt/backup/home/ /home/
tar -xzf backup.tar.gz -C /
sudo dd if=backup.img of=/dev/sda
```

## DESCRIPTION

Резервное копирование — важный аспект администрирования. Стратегия 3-2-1: 3 копии, на 2 разных носителях, 1 офсайт.

## BACKUP STRATEGIES

### 3-2-1 Rule

```
3 копии данных
2 разных типа носителя (HDD + USB)
1 копия в другом месте (офсайт)
```

### Full vs Incremental vs Differential

```
Full        — полная копия всех данных
Incremental — только изменения с последней резервной копии
Differential — только изменения с последней полной копии
```

## TOOLS

### rsync

Синхронизация и резервное копирование.

```bash
# Базовое копирование
sudo rsync -av /source/ /dest/

# С удалением файлов (опасно!)
sudo rsync -av --delete /source/ /dest/

# По сети
sudo rsync -av -e ssh /home/ user@remote:/backup/

# Только новые файлы
sudo rsync -av -u /source/ /dest/
```

### tar

Архивирование.

```bash
# Создать архив
tar -czf backup.tar.gz /home/

# Без сжатия
tar -cvf backup.tar /home/

# Список содержимого
tar -tzf backup.tar.gz

# Распаковать
tar -xzf backup.tar.gz -C /
```

### dd

Копирование диска (опасно!).

```bash
# Копировать диск
sudo dd if=/dev/sda of=backup.img bs=4M

# Восстановить из образа
sudo dd if=backup.img of=/dev/sda bs=4M

# Показывать прогресс
sudo dd if=/dev/sda of=backup.img bs=4M status=progress
```

## BACKUP PLAN

### Ежедневно

```bash
# Архив важных файлов
tar -czf ~/backups/daily-$(date +%Y%m%d).tar.gz ~/Documents ~/Photos
```

### Еженедельно

```bash
# Полная резервная копия домашней директории
sudo rsync -av /home/ /mnt/weekly-backup/
```

### Ежемесячно

```bash
# Образ диска
sudo dd if=/dev/sda of=/mnt/offsite/monthly-$(date +%Y%m).img bs=4M
```

## RECOVERY

### Восстановление файлов

```bash
# Из tar архива
tar -xzf backup.tar.gz /path/to/file

# Из rsync резервной копии
sudo rsync -av /mnt/backup/home/username/file /home/username/
```

### Восстановление диска

```bash
# Из образа dd
sudo dd if=backup.img of=/dev/sda bs=4M

# Проверить целостность после восстановления
sudo fsck /dev/sda1
```

### Восстановление системы

```bash
# Загрузиться с Live USB
# Примонтировать разделы
sudo mount /dev/sda1 /mnt

# Восстановить данные
sudo rsync -av /mnt/backup/ /mnt/
```

## TESTING

### Тестировать резервные копии

```bash
# Проверить tar архив
tar -tzf backup.tar.gz

# Распаковать в тестовую директорию
tar -xzf backup.tar.gz -C /tmp/test

# Проверить размер файлов
du -sh /tmp/test

# Сравнить оригинал с восстановленным
diff -r /home /tmp/test/home
```

### Тестировать восстановление

```bash
# На виртуальной машине
# Восстановить диск из образа
sudo dd if=backup.img of=/dev/sda bs=4M

# Загрузиться и проверить
```

## IMPORTANT FILES

```
/etc/               — конфигурация
/home/              — пользовательские данные
/var/www/           — веб-сервер
/var/lib/mysql/     — БД MySQL
/opt/               — дополнительное ПО
```

## COMMON MISTAKES

- ❌ Не тестировать восстановление
- ❌ Хранить копию рядом с оригиналом
- ❌ Забыть про шифрование при отправке офсайт
- ❌ Не архивировать логи и конфиги

## KEY TAKEAWAYS

- **3-2-1 Rule** — основа резервного копирования
- **Тестировать** — обязательно тестировать восстановление
- **Автоматизировать** — использовать cron
- **Шифровать** — особенно офсайт
- **Документировать** — что и когда резервируется

## SEE ALSO

- [[./01-what-is-systemd.md|What is systemd]]
- [[./02-units-services.md|Units and Services]]
- [[./03-package-management-advanced.md|Package Management]]
- [[./05-system-monitoring.md|System Monitoring]]
- [[./README.md|systemd README]]