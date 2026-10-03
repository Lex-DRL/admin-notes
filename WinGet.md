https://www.youtube.com/watch?v=KIJ76bjBr0Q

# Проверка, есть ли

```cmd
winget --version
```

# Установка, если нет

**НЕ в cmd, а именно в PowerShell от админа**

ПКМ по Пуск -> PowerShell (терминал в W11) от админа. Там:
```
Install-Module Microsoft.WinGet.Client -Force
Import-Module Microsoft.WinGet.Client
Repair-WinGetPackageManager -AllUsers -Force
```
# Команды - список

Список установленных программ:
	winget list

Поиск программы:
	winget search [название]

Список программ, у которых есть обновы:
	winget upgrade

# Обновление

Обновление программы:
	winget upgrade [ID(s)]

Если нужно именно через мастер:
	winget upgrade -i --allow-reboot --force [ID(s)]

# Установка

Установка программы:
	winget install [ID(s)]

Если нужно именно через мастер:
	winget install -i --allow-reboot --force [ID(s)]

# Управление пинами - т.е., автоматически НЕобновляемыми

Список запиненных:
	winget pin list

Запинить:
	winget pin add --exact [ID(s)]

# Удаление

Удаление программы
	winget uninstall [ID(s)]

# ОСТОРОЖНО!

Обновление **ВСЕХ** программ:
	`winget upgrade --all`
