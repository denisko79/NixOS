**Вот перевод и комментарии к вашему тексту. Я сохранил структуру оригинала, добавил пояснения там, где это может быть полезно, и убрал избыточные или устаревшие детали (например, упоминание `environment.pathsToLink`, которое в современных версиях NixOS считается устаревшим).**

---

## i3

**Breeze-preferences-other.png**  
*Эта статья или раздел требует расширения.*  
**Причина:** Статья является заготовкой. *(Обсуждение на Talk:I3#)*  
См. метастраницу статьи в вики для рекомендаций по внесению вклада.

**i3** — это тайлинговый оконный менеджер для X11.

---

### Включение i3 в NixOS

Чтобы использовать i3, установите параметр `services.xserver.windowManager.i3.enable = true`.

Пример конфигурации:

**Breeze-text-x-plain.png**  
`/etc/nixos/configuration.nix`

```nix
{ config, pkgs, ... }:

{
  # ...
  services.xserver = {
    enable = true;

    desktopManager.xterm.enable = false;  # отключаем xterm, если не нужен

    windowManager.i3 = {
      enable = true;
      extraPackages = with pkgs; [
        dmenu          # лаунчер приложений
        i3status       # стандартная панель состояния
        i3blocks       # альтернатива i3status
      ];
    };
  };

  services.displayManager.defaultSession = "none+i3";

  programs.i3lock.enable = true;  # блокировщик экрана по умолчанию
}
```

> **💡 Совет:** После внесения изменений в `configuration.nix` выполните:
> ```bash
> sudo nixos-rebuild switch
> ```

---

### Использование с Home Manager

Если вы используете **Home Manager**, основная часть настройки i3 переносится туда.

**Breeze-text-x-plain.png**  
`~/.config/nixpkgs/home.nix`

```nix
# В configuration.nix всё ещё нужно включить X-сервер и сессию:
services.xserver = {
  enable = true;
  windowManager.i3.enable = true;
};
services.displayManager.defaultSession = "none+i3";

# Для корректной работы блокировщиков экрана (опционально):
security.pam.services = {
  i3lock.enable = true;
  # i3lock-color, xlock, xscreensaver — включайте по необходимости
};
```

А в `home.nix`:

```nix
xsession.windowManager.i3 = {
  enable = true;
  package = pkgs.i3-gaps;  # можно использовать i3-gaps вместо vanilla i3
  config = {
    modifier = "Mod4";  # обычно это клавиша Super/Windows
    gaps = {
      inner = 10;
      outer = 5;
    };
  };
};
```

> 📌 См. также: [пример конфигурации от Srid](https://github.com/srid/nix-config/blob/master/nix/home/i3.nix)

---

### Использование с десктоп-менеджером

i3 — только оконный менеджер. Он **не управляет**:
- настройками мультимониторности,
- горячими клавишами мультимедиа,
- темами GTK и т.д.

Эти функции обычно предоставляет **десктоп-менеджер**. Например, можно использовать **XFCE** *только как десктоп-менеджер*, а не как оконный менеджер. Подробнее: [Xfce#Using_as_a_desktop_manager_and_not_a_window_manager](https://nixos.wiki/wiki/Xfce#Using_as_a_desktop_manager_and_not_a_window_manager).

---

### Советы и хитрости

#### i3blocks

По умолчанию i3blocks ищет скрипты в `/etc/i3blocks/`, но в NixOS исполняемые файлы находятся в `/nix/store/.../libexec/i3blocks/`.

Раньше рекомендовалось использовать `environment.pathsToLink = [ "/libexec" ];`, но **эта опция устарела**. Лучше явно указывать путь в конфиге i3blocks:

**`~/.config/i3/i3blocks.conf`**
```ini
[battery]
label=⚡
command=/run/current-system/sw/libexec/i3blocks/battery
interval=10
instance=1
```

> ✅ Путь `/run/current-system/sw/libexec/` автоматически доступен, если пакет установлен через `systemPackages` или `extraPackages`.

---

#### DConf

Если настройки GTK-приложений (например, размер диалоговых окон в Firefox) **не сохраняются**, включите DConf:

```nix
programs.dconf.enable = true;
```

---

#### Lxappearance

Для удобного выбора тем и иконок установите `lxappearance`:

```nix
environment.systemPackages = with pkgs; [
  lxappearance
  # ... другие пакеты
];
```

Запускайте через терминал: `lxappearance`.

---

#### Согласование тем GTK2 и GTK3

Иногда GTK2-приложения игнорируют тему, заданную для GTK3. Причина — отсутствие файла `~/.gtkrc-2.0`.

Создайте его вручную:

**`~/.gtkrc-2.0`**
```ini
gtk-theme-name="Sierra-compact-light"
gtk-icon-theme-name="ePapirus"
gtk-font-name="Ubuntu 11"
gtk-cursor-theme-name="Deepin"
gtk-cursor-theme-size=0
gtk-toolbar-style=GTK_TOOLBAR_BOTH
gtk-toolbar-icon-size=GTK_ICON_SIZE_LARGE_TOOLBAR
gtk-button-images=1
gtk-menu-images=1
gtk-enable-event-sounds=1
gtk-enable-input-feedback-sounds=1
gtk-xft-antialias=1
gtk-xft-hinting=1
gtk-xft-hintstyle="hintfull"
gtk-xft-rgba="rgb"
gtk-modules="gail:atk-bridge"
```

> ⚠️ Замените названия тем и шрифтов на те, что у вас установлены.

---

#### Обои рабочего стола

Если файл `~/.background-image` существует, он будет использован как обои.

Дополнительные настройки:
- `services.xserver.desktopManager.wallpaper.combineScreens` — объединять экраны при растяжении.
- `services.xserver.desktopManager.wallpaper.mode` — режим отображения (`stretch`, `center`, `fill` и т.д.).

> 💡 Альтернатива: использовать `feh --bg-scale ~/.background-image` в автозапуске i3.

---

#### i3status-rust с Home Manager

Home Manager может генерировать конфигурацию для `i3status-rust`, но **не подключает её автоматически** к i3. Нужно явно указать `statusCommand`.

**`~/.config/nixpkgs/home.nix`**
```nix
xsession.windowManager.i3 = {
  enable = true;
  config = {
    bars = [
      {
        position = "top";
        statusCommand = "${pkgs.i3status-rust}/bin/i3status-rs ${config.home.homeDirectory}/.config/i3status-rust/config-top.toml";
      }
    ];
  };
};

programs.i3status-rust = {
  enable = true;
  bars.top = {
    blocks = [
      {
        block = "time";
        interval = 60;
        format = " $timestamp.datetime('%a %d/%m %k:%M %p') ";
      }
    ];
  };
};
```

> 🔍 Убедитесь, что путь к конфигурационному файлу совпадает с тем, который генерирует Home Manager.

---

