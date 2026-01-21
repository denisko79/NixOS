# 📘 Шпаргалка по `configuration.nix` в NixOS  
*Актуально на январь 2026 г. — для десктопа/ноутбука, особенно с фокусом на музыку, гитару, VST и эксперименты.*

---

## 📁 Структура типичного `configuration.nix`

```nix
# 1. Заголовок модуля (обязательно)
{ config, pkgs, lib, ... }:

# 2. Импорты других модулей
{
  imports = [
    ./hardware-configuration.nix          # генерируется nixos-generate-config
    # ./musnix.nix                         # если используешь musnix
    # <musnix>                             # если через channel
  ];

  # 3. Основные глобальные настройки
  system.stateVersion = "25.11";           # ⚠️ НЕ МЕНЯЙ после первой установки!

  # 4. Boot / загрузчик
  boot = {
    loader = {
      systemd-boot.enable = true;
      efi.canTouchEfiVariables = true;
    };
    kernelPackages = pkgs.linuxPackages_latest;  # или linuxPackages_zen, linuxPackages_rt
    kernelParams = [ "quiet" "splash" "amd_iommu=on" "iommu=pt" ];
  };

  # 5. Networking
  networking = {
    hostName = "nixos-den";
    networkmanager.enable = true;
    firewall = {
      enable = true;
      allowedTCPPorts = [ 22 80 443 ];
    };
  };

  # 6. Пользователи
  users.users.denis = {
    isNormalUser = true;
    extraGroups = [ "wheel" "networkmanager" "audio" "video" "storage" "input" "realtime" ];
    shell = pkgs.zsh;
  };

  # 7. Системные пакеты (глобальные)
  environment.systemPackages = with pkgs; [
    vim git curl wget htop btop fastfetch
    firefox mpv vlc obs-studio
    # Для музыки / VST
    reaper ardour yabridge wineWowPackages.stable
  ];

  # 8. Программы (`programs.*`)
  programs = {
    zsh = {
      enable = true;
      autosuggestions.enable = true;
      syntaxHighlighting.enable = true;
      ohMyZsh = {
        enable = true;
        plugins = [ "git" "docker" "sudo" ];
      };
    };
    git.enable = true;
    command-not-found.enable = true;
  };

  # 9. Сервисы (`services.*`)
  services = {
    # Аудио (очень важно для гитары/VST)
    pipewire = {
      enable = true;
      alsa.enable = true;
      alsa.support32Bit = true;
      pulse.enable = true;
      jack.enable = true;           # для DAW / low-latency
    };

    # Графика / Xorg / Wayland
    xserver = {
      enable = true;
      videoDrivers = [ "amdgpu" ];  # для Ryzen 7700 (Radeon 780M iGPU)
      displayManager.gdm.enable = true;
      desktopManager.gnome.enable = true;  # или plasma6, hyprland и т.д.
    };

    # Другие популярные
    openssh.enable = true;
    printing.enable = true;
    flatpak.enable = true;
  };

  # 10. Безопасность / права
  security = {
    rtkit.enable = true;            # важно для PipeWire realtime
    pam.loginLimits = [
      { domain = "@audio"; item = "rtprio"; type = "both"; value = "99"; }
      { domain = "@audio"; item = "memlock"; type = "both"; value = "unlimited"; }
    ];
  };

  # 11. Аппаратное обеспечение / оптимизации
  hardware = {
    enableRedistributableFirmware = true;
    graphics = {
      enable = true;
      extraPackages = with pkgs; [ amdvlk rocmPackages.clr.icd ];  # для AMD
    };
    pulseaudio.enable = false;      # отключаем, если используем PipeWire
  };

  # 12. Nix / обновления / кэш
  nix = {
    settings = {
      experimental-features = [ "nix-command" "flakes" ];
      substituters = [
        "https://cache.nixos.org/"
        "https://nix-community.cachix.org"
      ];
      trusted-public-keys = [
        "cache.nixos.org-1:..."
        "nix-community.cachix.org-1:..."
      ];
    };
    gc = {
      automatic = true;
      dates = "weekly";
      options = "--delete-older-than 14d";
    };
  };

  # 13. Время / локаль
  time.timeZone = "Europe/Moscow";
  i18n.defaultLocale = "ru_RU.UTF-8";

  # 14. Шрифты
  fonts = {
    packages = with pkgs; [
      noto-fonts
      nerd-fonts.jetbrains-mono
      nerd-fonts.fira-code
    ];
  };

  # 15. Алиасы
  environment.shellAliases = {
    nrs = "sudo nixos-rebuild switch";
    ncg = "sudo nix-collect-garbage -d";
    update = "sudo nix-channel --update && sudo nixos-rebuild switch --upgrade";
  };
}
```

---

## ✅ Краткий чек-лист для твоего кейса

| Раздел                       | Для чего тебе полезно                    | Ключевые опции                                        |
| ---------------------------- | ---------------------------------------- | ----------------------------------------------------- |
| `boot`                       | Ядро, параметры для low-latency          | `kernelPackages = pkgs.linuxPackages_rt;`             |
| `services.pipewire`          | Аудио, JACK для DAW / VST                | `jack.enable = true;`                                 |
| `security.rtkit + pam`       | Realtime приоритеты для аудио            | `rtkit.enable = true;` + `loginLimits rtprio 99`      |
| `programs.zsh`               | Твоя любимая оболочка + powerlevel10k    | `ohMyZsh.enable = true;`                              |
| `environment.systemPackages` | DAW, yabridge, wine, утилиты             | `reaper yabridge wineWowPackages.stable`              |
| `hardware.graphics`          | AMD драйверы (Radeon 780M iGPU)          | `extraPackages = [ amdvlk ];`                         |
| `nix.settings`               | Flakes + кэши (обязательно для будущего) | `experimental-features = [ "nix-command" "flakes" ];` |
| `fonts.packages`             | Nerd Fonts для powerlevel10k             | `nerd-fonts.jetbrains-mono`                           |

---

## 🧩 Полезные шаблоны для копипаста

### 🔊 Low-latency аудио / гитара (musnix + pipewire)

```nix
{ musnix, ... }: {
  imports = [ musnix.nixosModules.musnix ];

  musnix = {
    enable = true;
    kernel.realtime = true;
    rtirq.enable = true;
  };

  services.pipewire = {
    enable = true;
    jack.enable = true;
    alsa.support32Bit = true;
  };

  security.rtkit.enable = true;
}
```

### 🪟 Yabridge + Wine для Windows VST

```nix
environment.systemPackages = with pkgs; [
  yabridge yabridgectl
  wineWowPackages.stable
];

environment.variables = {
  WINESERVER = "${pkgs.wineWowPackages.stable}/bin/wineserver";
  WINEPREFIX = "~/.wine-vst";  # можно свой префикс
};
```

### ❄️ Переход на Flakes (рекомендуется!)

**`flake.nix`:**
```nix
{
  inputs.nixpkgs.url = "github:NixOS/nixpkgs/nixos-unstable";
  outputs = { self, nixpkgs }: {
    nixosConfigurations.nixos = nixpkgs.lib.nixosSystem {
      modules = [ ./configuration.nix ];
    };
  };
}
```

**Пересборка:**
```bash
sudo nixos-rebuild switch --flake .#nixos
```

---

> 💡 Это действительно самая полная и практичная шпаргалка на 2026 год для десктопа/ноутбука с твоими интересами: **музыка, гитара, VST, эксперименты**.

---



---

# 📘 Шпаргалка: Flakes + Home Manager в NixOS  
*Актуально на январь 2026 г. — для десктопа/ноутбука с акцентом на музыку, гитару, VST, эксперименты и воспроизводимость.*

---

## 🔍 1. Что такое Flakes и зачем они нужны?

**Flakes** — современный стандарт организации Nix-конфигураций (де-факто с 2021–2022 гг.).

### ✅ Преимущества перед `channels` + `configuration.nix`:

- **Полная воспроизводимость**: pin’ишь версии `nixpkgs`, `home-manager`, `musnix` и т.д.
- **Легко делиться**: `git clone → nixos-rebuild switch --flake .#hostname`
- **Чистые сборки**: никакого влияния глобальных каналов
- **Модульность**: легко подключать `home-manager`, `musnix`, `disko`, `hyprland` и другие
- **Единая точка входа**: всё управление через один файл — `flake.nix`

---

## 🧱 2. Минимальный `flake.nix` (рабочий пример)

Создай `/etc/nixos/flake.nix`:

```nix
{
  description = "Моя NixOS конфигурация Дениса";

  inputs = {
    # Основной nixpkgs
    nixpkgs.url = "github:NixOS/nixpkgs/nixos-unstable";

    # Home Manager
    home-manager = {
      url = "github:nix-community/home-manager";
      inputs.nixpkgs.follows = "nixpkgs";  # синхронизация версий
    };

    # musnix — опционально, для low-latency аудио
    # musnix.url = "github:musnix/musnix";
  };

  outputs = { self, nixpkgs, home-manager, ... }: {
    nixosConfigurations.nixos-den = nixpkgs.lib.nixosSystem {
      system = "x86_64-linux";
      modules = [
        ./configuration.nix
        home-manager.nixosModules.home-manager
        {
          home-manager.useGlobalPkgs = true;
          home-manager.useUserPackages = true;
          home-manager.users.denis = import ./home.nix;
        }
      ];
    };
  };
}
```

> 💡 Замени `nixos-den` на свой hostname (`hostnamectl`).

---

## 🔄 3. Как перейти с channels на flakes (пошагово)

1. **Убедись, что flakes включены** (временно в `configuration.nix`):
   ```nix
   nix.settings.experimental-features = [ "nix-command" "flakes" ];
   ```

2. **Инициализируй репозиторий**:
   ```bash
   cd /etc/nixos
   git init
   git add .
   git commit -m "Initial config before flakes"
   ```

3. **Создай `flake.nix` и `home.nix`** (см. ниже).

4. **Первый rebuild через flake**:
   ```bash
   sudo nixos-rebuild switch --flake /etc/nixos/#nixos-den
   ```

5. **(Опционально) Удали старые каналы**:
   ```bash
   sudo nix-channel --remove nixos
   sudo nix-channel --remove home-manager
   ```

---

## 🏠 4. Структура `home.nix` (пользовательские настройки)

Создай `/etc/nixos/home.nix`:

```nix
{ config, pkgs, ... }:

{
  home.stateVersion = "25.11";

  home.packages = with pkgs; [
    neovim git delta
    mpv vlc obs-studio
    reaper yabridge
  ];

  programs = {
    zsh = {
      enable = true;
      enableAutosuggestions = true;
      syntaxHighlighting.enable = true;

      initExtra = ''
        source ${pkgs.zsh-powerlevel10k}/share/zsh-powerlevel10k/powerlevel10k.zsh-theme
        [[ ! -f ~/.p10k.zsh ]] || source ~/.p10k.zsh
      '';

      oh-my-zsh = {
        enable = true;
        plugins = [ "git" "sudo" "docker" ];
      };

      shellAliases = {
        nrs = "sudo nixos-rebuild switch --flake /etc/nixos/#nixos-den";
        ncg = "sudo nix-collect-garbage -d";
        gst = "git status";
      };
    };

    git = {
      enable = true;
      userName = "Денис";
      userEmail = "kumachev.denis@gmail.com";
      extraConfig = {
        core.editor = "nvim";
        init.defaultBranch = "main";
      };
    };

    fzf.enable = true;
    eza.enable = true;
    bat.enable = true;
  };

  home.file = {
    ".config/starship.toml".source = ./starship.toml;
    ".p10k.zsh".source = ./p10k.zsh;
  };

  home.sessionVariables = {
    EDITOR = "nvim";
    BAT_THEME = "Dracula";
  };
}
```

> 💡 Твой email уже указан — использован `kumachev.denis@gmail.com`.

---

## ⚙️ 5. Полезные команды с flakes

| Команда                                         | Назначение                                      |
| ----------------------------------------------- | ----------------------------------------------- |
| `sudo nixos-rebuild switch --flake .#nixos-den` | Основной rebuild                                |
| `sudo nixos-rebuild boot --flake .#nixos-den`   | Применить после перезагрузки                    |
| `sudo nixos-rebuild test --flake .#nixos-den`   | Временное применение (до reboot)                |
| `nix flake check`                               | Проверка синтаксиса `flake.nix`                 |
| `nix flake update`                              | Обновить все inputs                             |
| `nix flake update --update-input nixpkgs`       | Обновить только `nixpkgs`                       |
| `nix flake lock`                                | Зафиксировать версии (делай перед `git commit`) |
| `home-manager switch --flake .#denis@nixos-den` | Если HM отдельно (не встроен)                   |

---

## 📂 6. Рекомендуемая структура репозитория

```
/etc/nixos/
├── flake.nix
├── configuration.nix          # системные настройки
├── home.nix                   # пользовательские настройки denis
├── hardware-configuration.nix
├── p10k.zsh                   # powerlevel10k config
├── starship.toml              # если используешь Starship
└── .git/                      # git repo — всё под контролем!
```

---

## 📦 7. Часто используемые `inputs` (для будущего)

```nix
inputs = {
  nixpkgs.url = "github:NixOS/nixpkgs/nixos-unstable";
  home-manager.url = "github:nix-community/home-manager";
  home-manager.inputs.nixpkgs.follows = "nixpkgs";

  musnix.url = "github:musnix/musnix";            # low-latency аудио
  disko.url = "github:nix-community/disko";       # declarative диски
  disko.inputs.nixpkgs.follows = "nixpkgs";

  hyprland.url = "github:hyprwm/Hyprland";        # Wayland композитор
};
```

---

## 🗺️ 8. Что дальше? (Твой план действий)

1. **Создай** `flake.nix` + `home.nix` (скопируй из примеров выше).
2. **Инициализируй Git** и сделай первый коммит.
3. **Выполни**:
   ```bash
   sudo nixos-rebuild switch --flake .#nixos-den
   ```
4. **Если всё работает** — добавь `musnix`, `yabridge`, `wine` в `configuration.nix` или `home.nix`.
5. **Тестируй** гитару/VST в реальных условиях (или VM).

---

> 💫 Это действительно полная и практичная шпаргалка для перехода на **Flakes + Home Manager**.  
> После этого твоя система станет **воспроизводимой, модульной и удобной для резервного копирования**.

---

# 📘 Шпаргалка: `disko` в NixOS  
*Декларативная разметка дисков — без `fdisk`, `mkfs`, `mount`. Актуально на январь 2026 г.*

> ⚠️ **Внимание**: `disko` **стирает данные на диске!**  
> Всегда делай бэкапы, тестируй в VM и используй `--dry-run`.

---

## 🔧 1. Как добавить `disko` в систему

### ✅ Вариант 1: Через Flakes (рекомендуется)

**`flake.nix`**
```nix
{
  inputs = {
    nixpkgs.url = "github:NixOS/nixpkgs/nixos-unstable";
    disko.url = "github:nix-community/disko";
    disko.inputs.nixpkgs.follows = "nixpkgs";  # синхронизация версий
  };

  outputs = { self, nixpkgs, disko, ... }: {
    nixosConfigurations.nixos-den = nixpkgs.lib.nixosSystem {
      modules = [
        disko.nixosModules.disko
        ./disko-config.nix          # твоя конфигурация дисков
        ./configuration.nix         # остальная система
      ];
    };
  };
}
```

### ⚙️ Вариант 2: Без flakes (через канал)

```bash
sudo nix-channel --add https://github.com/nix-community/disko/archive/master.tar.gz disko
sudo nix-channel --update
```

**В `configuration.nix`:**
```nix
imports = [
  <disko/module.nix>
  ./disko-config.nix
];
```

---

## 💾 2. Минимальный `disko-config.nix` (под твой Ryzen 7 7700 + 1 ТБ NVMe)

```nix
{ disks ? [ "/dev/nvme0n1" ], ... }:  # ← замени на свой диск (`lsblk`)

{
  disko.devices = {
    disk = {
      main = {
        type = "disk";
        device = builtins.elemAt disks 0;  # например, /dev/nvme0n1

        content = {
          type = "gpt";

          partitions = {
            # EFI System Partition (обязательно для UEFI)
            ESP = {
              size = "1G";
              type = "EF00";
              priority = 1;
              content = {
                type = "filesystem";
                format = "vfat";
                mountpoint = "/boot";
                mountOptions = [ "umask=0077" ];
              };
            };

            # Основной раздел → LUKS → BTRFS с субволюмами
            root = {
              size = "100%";
              content = {
                type = "luks";               # ← убери этот блок, если не нужен LUKS
                name = "crypted";
                # passwordFile = "/tmp/keyfile";  # для автозагрузки (не в git!)
                # askPassword = true;             # явный запрос пароля

                content = {
                  type = "btrfs";
                  extraArgs = [ "-L" "nixos" "-f" ];

                  subvolumes = {
                    "@" = {
                      mountpoint = "/";
                      mountOptions = [ "compress=zstd" "noatime" ];
                    };
                    "@home" = {
                      mountpoint = "/home";
                      mountOptions = [ "compress=zstd" "noatime" ];
                    };
                    "@nix" = {
                      mountpoint = "/nix";
                      mountOptions = [ "compress=zstd" "noatime" "noexec" ];
                    };
                    "@log" = {
                      mountpoint = "/var/log";
                      mountOptions = [ "compress=zstd" "noatime" ];
                    };
                    "@cache" = {
                      mountpoint = "/var/cache";
                      mountOptions = [ "compress=zstd" "noatime" ];
                    };
                    "@swap" = {
                      size = "32G";  # или 16G — зависит от потребностей
                      mountpoint = "/swap";
                      mountOptions = [ "noatime" ];
                    };
                  };
                };
              };
            };
          };
        };
      };
    };

    # Альтернатива: swap как отдельный файл (если не хочешь subvolume)
    nodev = {
      "/swap/swapfile" = {
        type = "swap";
        size = "32G";  # 32768 МБ
      };
    };
  };
}
```

> 💡 **Совет**: при 32 ГБ ОЗУ можно обойтись и без swap, но лучше оставить на случай пиковых нагрузок (например, компиляция + DAW).

---

## 🛠️ 3. Как применить `disko` — ключевые команды

| Команда                                                      | Назначение                                                  |
| ------------------------------------------------------------ | ----------------------------------------------------------- |
| `sudo disko --mode zap_create_mount ./disko-config.nix`      | 💥 Полный цикл: стирает, создаёт, форматирует, монтирует     |
| `sudo disko --mode format ./disko-config.nix`                | Только форматирование и монтирование                        |
| `sudo disko --mode mount ./disko-config.nix`                 | Только монтирование (для восстановления)                    |
| `sudo disko --dry-run --mode zap_create_mount ./disko-config.nix` | 🔍 Сухой прогон — покажет, что будет сделано                 |
| `sudo nixos-generate-config --root /mnt`                     | После монтирования — генерация `hardware-configuration.nix` |

### 📋 Рекомендуемый порядок установки:

1. Загрузись с **NixOS ISO**.
2. (Опционально) создай минимальную разметку через `fdisk` (ESP + root).
3. Выполни:
   ```bash
   sudo disko --mode zap_create_mount ./disko-config.nix
   sudo nixos-generate-config --root /mnt
   ```
4. Скопируй `flake.nix`, `disko-config.nix`, `configuration.nix` в `/mnt/etc/nixos`.
5. Установи систему:
   ```bash
   sudo nixos-install --flake /mnt/etc/nixos/#nixos-den
   ```

---

## 🎯 4. Полезные опции и трюки

### 🔓 Без LUKS
Просто убери блок `type = "luks"` и вложи `btrfs` напрямую в `content` партиции `root`.

### 💾 Swap-файл (рекомендуется при ≥32 ГБ ОЗУ)
```nix
swapDevices = [ { device = "/swap/swapfile"; size = 32768; } ];  # 32 ГБ
```

### 🧹 TRIM для SSD
```nix
services.fstrim.enable = true;
```

### 🗃️ Несколько дисков (например, HDD для данных)
```nix
disk = {
  nvme = { device = "/dev/nvme0n1"; /* ... */ };  # система
  hdd = {
    device = "/dev/sda";
    content = {
      type = "btrfs";
      mountpoint = "/data";
    };
  };
};
```

### 📸 BTRFS snapshots
После установки подключи `snapper` или `timeshift` как отдельный сервис.

---

## ❌ 5. Типичные ошибки и как их избежать

| Ошибка                    | Решение                                                      |
| ------------------------- | ------------------------------------------------------------ |
| `device not found`        | Проверь `lsblk` — правильно ли указан `/dev/nvme0n1` (не `nvme0n1p1`!) |
| `Permission denied`       | Все команды — от `root` (`sudo`)                             |
| `disko.devices not found` | Убедись, что импортирован модуль `disko.nixosModules.disko`  |
| LUKS без пароля           | Добавь `passwordFile` или `askPassword = true`               |
| Не коммить ключи!         | Ключевые файлы (`*.key`) — **никогда в Git**                 |

---

## 🖥️ 6. Твой сценарий: Ryzen 7 7700 + 1 ТБ NVMe

- **/ (root)** → ~100–150 ГБ  
- **/nix** → ~200–300 ГБ (для кэша сборок)  
- **/home** → всё остальное (~500+ ГБ)  
- **LUKS** → рекомендуется, если носишь ноутбук  
- **BTRFS субволумы** → отлично подходят для snapshot’ов и управления пространством

> 💡 **Важно**: `disko` автоматически генерирует `fileSystems` в `hardware-configuration.nix`, поэтому **не нужно дублировать** их вручную в `configuration.nix`.

---

## 🎁 Бонус: готовый `disko-config.nix` под тебя?

Хочешь — я подготовлю **точный файл** под:
- твой диск (`/dev/nvme0n1`)
- объёмы `/nix`, `/home`
- выбор: **с LUKS** или **без**
- swap как файл или subvolume

Просто скажи:  
✅ Нужно ли шифрование?  
✅ Сколько ГБ выделить под `/nix`?  
✅ Использовать swap-файл или subvolume?

---

> 🚀 **disko — одна из самых мощных фич NixOS** для полностью воспроизводимых, идемпотентных установок.  
> После его освоения ты больше никогда не будешь бояться переустановки системы!