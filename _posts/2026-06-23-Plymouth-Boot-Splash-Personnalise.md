---
title: "Plymouth : personnaliser le boot splash de ton Linux"
date: 2026-06-23 10:00 +0200
categories: ["Sysops"]
tags: ["plymouth", "linux", "boot", "customisation", "ubuntu", "dracut", "sysops", "thème"]
author: marvax
---

> Le boot screen par défaut d'Ubuntu c'est moche. Plymouth te permet de le remplacer par n'importe quel thème animé. Voici comment créer le tien, de zéro, sans casser ton système.

<!--more/>

## Plymouth en 30 secondes

Plymouth est le framework qui affiche le splash screen pendant le boot (et le prompt LUKS si disque chiffré). Il tourne en userspace dans l'initramfs.

```bash
# Vérifier si Plymouth est actif
plymouth --ping && echo "Plymouth actif" || echo "Plymouth inactif"

# Lister les thèmes installés
plymouth-set-default-theme -l

# Changer de thème
sudo plymouth-set-default-theme -R mon-theme
```

## Structure d'un thème

```
/usr/share/plymouth/themes/mon-theme/
├── mon-theme.plymouth      # Descripteur
├── mon-theme.script         # Script d'animation
└── images/
    ├── logo.png             # Logo (recommandé 200-400px)
    ├── background.png       # Fond d'écran (1920x1080)
    └── throbber/            # Animation de chargement
        ├── 0001.png
        ├── 0002.png
        └── ...              # 60 frames recommandées
```

## Le fichier .plymouth

```ini
[Plymouth Theme]
Name=Mon Theme
Description=Boot splash personnalisé
ModuleName=script

[script]
ImageDir=/usr/share/plymouth/themes/mon-theme/images
ScriptFile=/usr/share/plymouth/themes/mon-theme/mon-theme.script
```

## Le script d'animation

```javascript
// mon-theme.script — Plymouth script (syntaxe similaire à JS)

// Charger les images
logo.image = Image("logo.png");
background.image = Image("background.png");

// Sprite pour le throbber (animation de chargement)
throbber.sprite = Sprite();
throbber.image = Image("throbber/0001.png");
throbber.sprite.SetImage(throbber.image);

// Position du logo (centré)
logo.sprite = Sprite();
logo.sprite.SetImage(logo.image);
logo.sprite.SetX(Window.GetWidth() / 2 - logo.image.GetWidth() / 2);
logo.sprite.SetY(Window.GetHeight() / 2 - logo.image.GetHeight() / 2);

// Fond d'écran
background.sprite = Sprite();
background.sprite.SetImage(background.image);
background.sprite.SetX(0);
background.sprite.SetY(0);

// Barre de progression
progress_bar.sprite = Sprite();
progress_bar.image = Image("progress_bar.png");

// Animation du throbber
throbber_count = 60;
throbber_current = 0;

callback refresh_callback() {
    throbber_current++;
    if (throbber_current >= throbber_count)
        throbber_current = 0;
    
    filename = String("throbber/%04d.png").Form(throbber_current + 1);
    throbber.image = Image(filename);
    throbber.sprite.SetImage(throbber.image);
    
    return true;
}

// Afficher le message de boot
message_sprite = Sprite();
message_sprite.SetPosition(Window.GetWidth() / 2, Window.GetHeight() * 0.75, 1);

callback message_callback(text) {
    message_sprite.SetImage(text.Image);
}

// Prompt LUKS (pour disque chiffré)
callback display_normal_password_prompt(prompt) {
    message_sprite.SetPosition(Window.GetWidth() / 2, Window.GetHeight() * 0.75, 1);
    message_sprite.SetImage(prompt.Image);
}
```

## Générer le throbber

```bash
# Avec ImageMagick — créer 60 frames rotatives
for i in $(seq -w 1 60); do
    angle=$((i * 6))  # 360/60 = 6 degrés par frame
    convert logo.png -rotate $angle -resize 64x64 "throbber/$i.png"
done

# Ou avec FFmpeg à partir d'une vidéo
ffmpeg -i spinner.mp4 -vf "scale=64:64,fps=15" throbber/%04d.png
```

## Installation du thème

```bash
# Copier les fichiers
sudo cp -r mon-theme /usr/share/plymouth/themes/

# Définir comme thème par défaut + rebuild initramfs
sudo plymouth-set-default-theme -R mon-theme

# Sur Ubuntu avec dracut (26.04+)
sudo dracut -f

# Sur Ubuntu avec initramfs-tools (< 26.04)
sudo update-initramfs -u
```

## Le piège dracut (Ubuntu 26.04)

Ubuntu 26.04 utilise **dracut** au lieu d'initramfs-tools. Les hooks Plymouth fonctionnent différemment :

```bash
# Vérifier si le module plymouth est chargé
lsmod | grep plymouth

# Forcer l'inclusion dans dracut
# /etc/dracut.conf.d/plymouth.conf
force_drivers+=" plymouth "
```

## Thèmes populaires

```bash
# Installer des thèmes depuis les dépôts
sudo apt install plymouth-theme-*

# Ou télécharger
# - Plymouth Themes Manager (GUI)
# - github.com/search?q=plymouth+theme
```

## Prompt LUKS personnalisé

Si ton disque est chiffré, Plymouth affiche aussi le prompt de déchiffrement. Personnalise-le :

```javascript
// Dans le .script
callback display_normal_password_prompt(prompt) {
    password_sprite = Sprite();
    password_sprite.SetImage(prompt.Image);
    password_sprite.SetX(Window.GetWidth() / 2 - prompt.image.GetWidth() / 2);
    password_sprite.SetY(Window.GetHeight() * 0.75);
}
```

## Debug

```bash
# Tester un thème sans rebooter
sudo plymouthd --debug --tty=/dev/tty7
sudo plymouth --show-splash
sleep 5
sudo plymouth --quit

# Logs
journalctl -b | grep -i plymouth

# Vérifier l'initramfs
lsinitrd /boot/initrd.img-$(uname -r) | grep plymouth
```

---

*Références :*
- [Plymouth wiki](https://wiki.archlinux.org/title/Plymouth)
- [Plymouth scripting reference](https://www.freedesktop.org/wiki/Software/Plymouth/Scripts/)

*Tu veux un boot splash unique pour ton serveur ? [Contacte-moi](mailto:m4rv4x@protonmail.com).*
