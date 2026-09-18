#!/usr/bin/env bash

# ============================================================
# SISI STORE - PTERODACTYL THEME INSTALLER
# ============================================================

# ============================================================
# EDIT BAGIAN INI UNTUK GANTI TEMA
# ============================================================
THEME_URL="https://raw.githubusercontent.com/USERNAME/REPOSITORY/main/theme.zip"

# ============================================================
# CONFIG
# ============================================================
PANEL_DIR=""
BACKUP_DIR=""
TEMP_DIR=""

# ============================================================
# COLORS
# ============================================================
PURPLE='\033[38;5;141m'
NEON='\033[38;5;135m'
CYAN='\033[38;5;51m'
GREEN='\033[38;5;82m'
RED='\033[38;5;196m'
YELLOW='\033[38;5;220m'
WHITE='\033[97m'
GRAY='\033[90m'
RESET='\033[0m'

# ============================================================
# FUNCTIONS
# ============================================================

success() {
    echo -e "${GREEN}[✓]${RESET} $1"
}

error() {
    echo -e "${RED}[✗]${RESET} $1"
}

warning() {
    echo -e "${YELLOW}[!]${RESET} $1"
}

info() {
    echo -e "${PURPLE}[SISI]${RESET} $1"
}

die() {
    error "$1"
    cleanup
    exit 1
}

cleanup() {
    if [[ -n "$TEMP_DIR" && -d "$TEMP_DIR" ]]; then
        rm -rf "$TEMP_DIR"
    fi
}

trap cleanup EXIT

# ============================================================
# LOGO
# ============================================================

show_logo() {
    clear 2>/dev/null || true

    echo -e "${NEON}"
    cat <<'EOF'
███████╗██╗███████╗██╗
██╔════╝██║██╔════╝██║
███████╗██║███████╗██║
╚════██║██║╚════██║██║
███████║██║███████║██║
╚══════╝╚═╝╚══════╝╚═╝

███████╗████████╗ ██████╗ ██████╗ ███████╗
██╔════╝╚══██╔══╝██╔═══██╗██╔══██╗██╔════╝
███████╗   ██║   ██║   ██║██████╔╝█████╗
╚════██║   ██║   ██║   ██║██╔══██╗██╔══╝
███████║   ██║   ╚██████╔╝██║  ██║███████╗
╚══════╝   ╚═╝    ╚═════╝ ╚═╝  ╚═╝╚══════╝
EOF
    echo -e "${RESET}"
    echo -e "${GRAY}Pterodactyl Theme Installer${RESET}"
    echo
}

# ============================================================
# ROOT CHECK
# ============================================================

check_root() {
    if [[ "$EUID" -ne 0 ]]; then
        error "Harus pakai root!"
        echo
        echo "Jalankan:"
        echo
        echo "sudo bash <(curl -s URL_INSTALLER)"
        echo
        exit 1
    fi
}

# ============================================================
# COMMAND CHECK
# ============================================================

command_exists() {
    command -v "$1" >/dev/null 2>&1
}

# ============================================================
# DEPENDENCY
# ============================================================

install_dependencies() {
    local missing=()

    for cmd in curl unzip tar file; do
        if ! command_exists "$cmd"; then
            missing+=("$cmd")
        fi
    done

    if [[ "${#missing[@]}" -eq 0 ]]; then
        return
    fi

    info "Menginstall dependency: ${missing[*]}"

    if ! command_exists apt-get; then
        die "apt-get tidak ditemukan. Installer ini ditujukan untuk Ubuntu/Debian."
    fi

    apt-get update -y >/dev/null 2>&1 || \
        die "Gagal menjalankan apt update."

    apt-get install -y "${missing[@]}" >/dev/null 2>&1 || \
        die "Gagal menginstall dependency."

    success "Dependency berhasil dipasang."
}

# ============================================================
# OS DETECTION
# ============================================================

detect_os() {
    if [[ ! -f /etc/os-release ]]; then
        warning "Tidak dapat mendeteksi OS."
        return
    fi

    # shellcheck disable=SC1091
    source /etc/os-release

    echo -e "OS      : ${CYAN}${PRETTY_NAME:-Unknown}${RESET}"

    if [[ "$ID" == "ubuntu" ]]; then
        case "$VERSION_ID" in
            20.04|22.04|24.04)
                success "Ubuntu ${VERSION_ID} didukung."
                ;;
            *)
                warning "Ubuntu ${VERSION_ID} belum diuji oleh installer."
                ;;
        esac
    else
        warning "Installer ditujukan terutama untuk Ubuntu 20.04/22.04/24.04."
    fi
}

# ============================================================
# PANEL DETECTION
# ============================================================

find_panel() {
    local paths=(
        "/var/www/pterodactyl"
        "/var/www/panel"
        "/var/www/pterodactyl-panel"
    )

    for path in "${paths[@]}"; do
        if [[ -f "$path/artisan" && -d "$path/resources" ]]; then
            PANEL_DIR="$path"
            return 0
        fi
    done

    while IFS= read -r artisan; do
        local dir
        dir="$(dirname "$artisan")"

        if [[ -d "$dir/resources" ]]; then
            PANEL_DIR="$dir"
            return 0
        fi
    done < <(
        find /var/www \
            -maxdepth 5 \
            -type f \
            -name artisan \
            2>/dev/null
    )

    return 1
}

detect_panel() {
    info "Mendeteksi instalasi Pterodactyl..."

    if ! find_panel; then
        die "Instalasi Pterodactyl tidak ditemukan."
    fi

    success "Panel ditemukan: $PANEL_DIR"
}

# ============================================================
# VERSION
# ============================================================

detect_version() {
    PANEL_VERSION="Unknown"

    if [[ -f "$PANEL_DIR/artisan" ]] && command_exists php; then
        PANEL_VERSION="$(
            cd "$PANEL_DIR" &&
            php artisan --version 2>/dev/null |
            grep -oE '[0-9]+\.[0-9]+\.[0-9]+' |
            head -n1
        )"
    fi

    [[ -z "$PANEL_VERSION" ]] && PANEL_VERSION="Unknown"

    echo -e "Version : ${CYAN}${PANEL_VERSION}${RESET}"

    case "$PANEL_VERSION" in
        1.6.*)
            success "Pterodactyl 1.6.x terdeteksi."
            ;;
        1.7.*)
            success "Pterodactyl 1.7.x terdeteksi."
            ;;
        *)
            warning "Versi tidak terdeteksi sebagai 1.6.x/1.7.x."
            warning "Melanjutkan menggunakan mode kompatibilitas."
            ;;
    esac
}

# ============================================================
# BACKUP
# ============================================================

create_backup() {
    local date
    date="$(date '+%Y%m%d-%H%M%S')"

    BACKUP_DIR="/var/www/pterodactyl-backup-${date}"

    info "Membuat backup..."

    mkdir -p "$BACKUP_DIR" || \
        die "Tidak dapat membuat folder backup."

    cp -a "$PANEL_DIR/resources" "$BACKUP_DIR/" || {
        rm -rf "$BACKUP_DIR"
        die "Backup resources gagal."
    }

    [[ -f "$PANEL_DIR/package.json" ]] &&
        cp -a "$PANEL_DIR/package.json" "$BACKUP_DIR/"

    [[ -f "$PANEL_DIR/yarn.lock" ]] &&
        cp -a "$PANEL_DIR/yarn.lock" "$BACKUP_DIR/"

    [[ -f "$PANEL_DIR/package-lock.json" ]] &&
        cp -a "$PANEL_DIR/package-lock.json" "$BACKUP_DIR/"

    cat > "$BACKUP_DIR/backup-info.txt" <<EOF
SISI STORE Pterodactyl Theme Backup

Date: $(date)
Panel: $PANEL_DIR
Version: $PANEL_VERSION
Theme URL: $THEME_URL
EOF

    success "Backup dibuat:"
    echo -e "        ${CYAN}${BACKUP_DIR}${RESET}"
}

# ============================================================
# DOWNLOAD
# ============================================================

download_theme() {
    TEMP_DIR="$(mktemp -d /tmp/sisi-theme.XXXXXX)"

    THEME_FILE="$TEMP_DIR/theme"

    info "Download tema..."

    curl \
        --fail \
        --location \
        --show-error \
        --progress-bar \
        "$THEME_URL" \
        -o "$THEME_FILE" || {
            die "Gagal mendownload tema."
        }

    if [[ ! -s "$THEME_FILE" ]]; then
        die "File tema kosong."
    fi

    success "Tema berhasil didownload."
}

# ============================================================
# EXTRACT
# ============================================================

extract_theme() {
    THEME_EXTRACT="$TEMP_DIR/extracted"

    mkdir -p "$THEME_EXTRACT"

    info "Extract tema..."

    local file_type
    file_type="$(file -b "$THEME_FILE" 2>/dev/null || true)"

    if [[ "$file_type" == *"Zip archive"* ]]; then

        unzip -q "$THEME_FILE" -d "$THEME_EXTRACT" || \
            die "Extract ZIP gagal."

    elif [[ "$file_type" == *"gzip compressed"* ]]; then

        tar -xzf "$THEME_FILE" -C "$THEME_EXTRACT" || \
            die "Extract TAR.GZ gagal."

    elif [[ "$file_type" == *"tar archive"* ]]; then

        tar -xf "$THEME_FILE" -C "$THEME_EXTRACT" || \
            die "Extract TAR gagal."

    else
        die "Format tema tidak dikenali. Gunakan ZIP atau TAR.GZ."
    fi

    success "Tema berhasil diekstrak."
}

# ============================================================
# FIND THEME STRUCTURE
# ============================================================

find_theme_resources() {

    if [[ -d "$THEME_EXTRACT/resources" ]]; then
        THEME_RESOURCES="$THEME_EXTRACT/resources"
        return 0
    fi

    local found
    found="$(
        find "$THEME_EXTRACT" \
            -type d \
            -name resources \
            -print \
            -quit \
            2>/dev/null
    )"

    if [[ -n "$found" ]]; then
        THEME_RESOURCES="$found"
        return 0
    fi

    if [[ -d "$THEME_EXTRACT/views" ||
          -d "$THEME_EXTRACT/scripts" ||
          -d "$THEME_EXTRACT/js" ||
          -d "$THEME_EXTRACT/css" ]]; then

        THEME_RESOURCES="$THEME_EXTRACT"
        return 0
    fi

    return 1
}

# ============================================================
# INSTALL
# ============================================================

install_theme() {
    info "Memasang file tema..."

    if ! find_theme_resources; then
        die "Struktur tema tidak valid."
    fi

    cp -a "$THEME_RESOURCES/." "$PANEL_DIR/resources/" || \
        die "Gagal menyalin file tema."

    success "File tema berhasil dipasang."
}

# ============================================================
# BUILD
# ============================================================

build_panel() {
    cd "$PANEL_DIR" || \
        die "Tidak dapat masuk ke direktori panel."

    if ! command_exists php; then
        die "PHP tidak ditemukan."
    fi

    info "Mengaktifkan maintenance mode..."

    php artisan down >/dev/null 2>&1 || \
        warning "Maintenance mode gagal diaktifkan."

    local build_ok=false

    echo
    info "Mendeteksi build system..."

    if [[ -f "yarn.lock" ]] && command_exists yarn; then

        echo -e "Build   : ${CYAN}yarn build:production${RESET}"
        echo

        if yarn build:production; then
            build_ok=true
        fi

    elif [[ -f "package-lock.json" ]] && command_exists npm; then

        echo -e "Build   : ${CYAN}npm run build${RESET}"
        echo

        if npm run build; then
            build_ok=true
        fi

    elif command_exists yarn; then

        echo -e "Build   : ${CYAN}yarn build:production${RESET}"
        echo

        if yarn build:production; then
            build_ok=true
        fi

    elif command_exists npm; then

        echo -e "Build   : ${CYAN}npm run build${RESET}"
        echo

        if npm run build; then
            build_ok=true
        fi

    else
        php artisan up >/dev/null 2>&1 || true
        die "Yarn/npm tidak ditemukan."
    fi

    if [[ "$build_ok" != true ]]; then
        error "Build panel gagal."
        warning "Maintenance mode dibiarkan aktif untuk keamanan."
        warning "Gunakan backup berikut untuk restore:"
        echo -e "        ${CYAN}${BACKUP_DIR}${RESET}"
        exit 1
    fi

    success "Build selesai."

    php artisan optimize:clear >/dev/null 2>&1 || true

    info "Menonaktifkan maintenance mode..."

    php artisan up >/dev/null 2>&1 || \
        die "Gagal mengaktifkan kembali panel."

    success "Panel kembali online."
}

# ============================================================
# PERMISSION
# ============================================================

fix_permissions() {
    info "Memperbaiki permission..."

    chown -R www-data:www-data "$PANEL_DIR" || \
        die "Gagal memperbaiki ownership."

    success "Permission selesai."
}

# ============================================================
# RESTORE
# ============================================================

restore_backup() {

    echo
    echo -e "${NEON}SISI STORE - RESTORE BACKUP${RESET}"
    echo

    mapfile -t backups < <(
        find /var/www \
            -maxdepth 1 \
            -type d \
            -name 'pterodactyl-backup-*' \
            -print 2>/dev/null |
        sort -r
    )

    if [[ "${#backups[@]}" -eq 0 ]]; then
        warning "Backup tidak ditemukan."
        return
    fi

    local i=1

    for backup in "${backups[@]}"; do
        echo "[$i] $(basename "$backup")"
        ((i++))
    done

    echo
    read -rp "Pilih backup [1-${#backups[@]}]: " choice

    if ! [[ "$choice" =~ ^[0-9]+$ ]]; then
        error "Pilihan tidak valid."
        return
    fi

    if (( choice < 1 || choice > ${#backups[@]} )); then
        error "Backup tidak tersedia."
        return
    fi

    local selected="${backups[$((choice-1))]}"

    echo
    warning "Resources panel saat ini akan diganti dengan backup."
    read -rp "Lanjutkan? [y/N]: " confirm

    if [[ ! "$confirm" =~ ^[Yy]$ ]]; then
        info "Restore dibatalkan."
        return
    fi

    cd "$PANEL_DIR" || return

    php artisan down >/dev/null 2>&1 || true

    if [[ ! -d "$selected/resources" ]]; then
        php artisan up >/dev/null 2>&1 || true
        error "resources tidak ditemukan dalam backup."
        return
    fi

    rm -rf "$PANEL_DIR/resources"

    cp -a "$selected/resources" "$PANEL_DIR/" || {
        php artisan up >/dev/null 2>&1 || true
        error "Restore gagal."
        return
    }

    fix_permissions

    if [[ -f "yarn.lock" ]] && command_exists yarn; then
        yarn build:production || warning "Build setelah restore gagal."

    elif command_exists npm; then
        npm run build || warning "Build setelah restore gagal."
    fi

    php artisan optimize:clear >/dev/null 2>&1 || true
    php artisan up >/dev/null 2>&1 || true

    success "Backup berhasil direstore."
}

# ============================================================
# INSTALL PROCESS
# ============================================================

install_process() {

    echo -e "${PURPLE}"
    echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
    echo -e "${WHITE}          SISI STORE INSTALLER${RESET}"
    echo -e "${PURPLE}━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━${RESET}"
    echo

    check_root
    detect_os
    install_dependencies
    detect_panel
    detect_version

    echo
    echo -e "Panel : ${CYAN}${PANEL_DIR}${RESET}"
    echo -e "Theme : ${CYAN}${THEME_URL}${RESET}"
    echo

    read -rp "Lanjutkan instalasi? [y/N]: " confirm

    if [[ ! "$confirm" =~ ^[Yy]$ ]]; then
        info "Instalasi dibatalkan."
        exit 0
    fi

    echo

    create_backup
    download_theme
    extract_theme
    install_theme
    build_panel
    fix_permissions

    echo
    echo -e "${GREEN}"
    cat <<'EOF'
╔══════════════════════════════════════════════╗
║                                              ║
║          ✓ INSTALLASI BERHASIL              ║
║                                              ║
╚══════════════════════════════════════════════╝
EOF
    echo -e "${RESET}"

    echo -e "Backup tersimpan di:"
    echo -e "${CYAN}${BACKUP_DIR}${RESET}"
    echo
}

# ============================================================
# MAIN MENU
# ============================================================

main() {

    show_logo
    check_root

    echo -e "${WHITE}[1] Install / Update Theme${RESET}"
    echo -e "${WHITE}[2] Restore / Uninstall Theme${RESET}"
    echo -e "${WHITE}[3] Exit${RESET}"
    echo

    read -rp "Pilih [1-3]: " option

    case "$option" in

        1)
            install_process
            ;;

        2)
            detect_panel
            restore_backup
            ;;

        3)
            exit 0
            ;;

        *)
            error "Pilihan tidak valid."
            exit 1
            ;;

    esac
}

main#!/usr/bin/env bash

# ============================================================
# SISI STORE - PTERODACTYL THEME INSTALLER
# ============================================================

# ============================================================
# EDIT BAGIAN INI UNTUK GANTI TEMA
# ============================================================
THEME_URL="https://raw.githubusercontent.com/USERNAME/REPOSITORY/main/theme.zip"

# ============================================================
# CONFIG
# ============================================================
PANEL_DIR=""
BACKUP_DIR=""
TEMP_DIR=""

# ============================================================
# COLORS
# ============================================================
PURPLE='\033[38;5;141m'
NEON='\033[38;5;135m'
CYAN='\033[38;5;51m'
GREEN='\033[38;5;82m'
RED='\033[38;5;196m'
YELLOW='\033[38;5;220m'
WHITE='\033[97m'
GRAY='\033[90m'
RESET='\033[0m'

# ============================================================
# FUNCTIONS
# ============================================================

success() {
    echo -e "${GREEN}[✓]${RESET} $1"
}

error() {
    echo -e "${RED}[✗]${RESET} $1"
}

warning() {
    echo -e "${YELLOW}[!]${RESET} $1"
}

info() {
    echo -e "${PURPLE}[SISI]${RESET} $1"
}

die() {
    error "$1"
    cleanup
    exit 1
}

cleanup() {
    if [[ -n "$TEMP_DIR" && -d "$TEMP_DIR" ]]; then
        rm -rf "$TEMP_DIR"
    fi
}

trap cleanup EXIT

# ============================================================
# LOGO
# ============================================================

show_logo() {
    clear 2>/dev/null || true

    echo -e "${NEON}"
    cat <<'EOF'
███████╗██╗███████╗██╗
██╔════╝██║██╔════╝██║
███████╗██║███████╗██║
╚════██║██║╚════██║██║
███████║██║███████║██║
╚══════╝╚═╝╚══════╝╚═╝

███████╗████████╗ ██████╗ ██████╗ ███████╗
██╔════╝╚══██╔══╝██╔═══██╗██╔══██╗██╔════╝
███████╗   ██║   ██║   ██║██████╔╝█████╗
╚════██║   ██║   ██║   ██║██╔══██╗██╔══╝
███████║   ██║   ╚██████╔╝██║  ██║███████╗
╚══════╝   ╚═╝    ╚═════╝ ╚═╝  ╚═╝╚══════╝
EOF
    echo -e "${RESET}"
    echo -e "${GRAY}Pterodactyl Theme Installer${RESET}"
    echo
}

# ============================================================
# ROOT CHECK
# ============================================================

check_root() {
    if [[ "$EUID" -ne 0 ]]; then
        error "Harus pakai root!"
        echo
        echo "Jalankan:"
        echo
        echo "sudo bash <(curl -s URL_INSTALLER)"
        echo
        exit 1
    fi
}

# ============================================================
# COMMAND CHECK
# ============================================================

command_exists() {
    command -v "$1" >/dev/null 2>&1
}

# ============================================================
# DEPENDENCY
# ============================================================

install_dependencies() {
    local missing=()

    for cmd in curl unzip tar file; do
        if ! command_exists "$cmd"; then
            missing+=("$cmd")
        fi
    done

    if [[ "${#missing[@]}" -eq 0 ]]; then
        return
    fi

    info "Menginstall dependency: ${missing[*]}"

    if ! command_exists apt-get; then
        die "apt-get tidak ditemukan. Installer ini ditujukan untuk Ubuntu/Debian."
    fi

    apt-get update -y >/dev/null 2>&1 || \
        die "Gagal menjalankan apt update."

    apt-get install -y "${missing[@]}" >/dev/null 2>&1 || \
        die "Gagal menginstall dependency."

    success "Dependency berhasil dipasang."
}

# ============================================================
# OS DETECTION
# ============================================================

detect_os() {
    if [[ ! -f /etc/os-release ]]; then
        warning "Tidak dapat mendeteksi OS."
        return
    fi

    # shellcheck disable=SC1091
    source /etc/os-release

    echo -e "OS      : ${CYAN}${PRETTY_NAME:-Unknown}${RESET}"

    if [[ "$ID" == "ubuntu" ]]; then
        case "$VERSION_ID" in
            20.04|22.04|24.04)
                success "Ubuntu ${VERSION_ID} didukung."
                ;;
            *)
                warning "Ubuntu ${VERSION_ID} belum diuji oleh installer."
                ;;
        esac
    else
        warning "Installer ditujukan terutama untuk Ubuntu 20.04/22.04/24.04."
    fi
}

# ============================================================
# PANEL DETECTION
# ============================================================

find_panel() {
    local paths=(
        "/var/www/pterodactyl"
        "/var/www/panel"
        "/var/www/pterodactyl-panel"
    )

    for path in "${paths[@]}"; do
        if [[ -f "$path/artisan" && -d "$path/resources" ]]; then
            PANEL_DIR="$path"
            return 0
        fi
    done

    while IFS= read -r artisan; do
        local dir
        dir="$(dirname "$artisan")"

        if [[ -d "$dir/resources" ]]; then
            PANEL_DIR="$dir"
            return 0
        fi
    done < <(
        find /var/www \
            -maxdepth 5 \
            -type f \
            -name artisan \
            2>/dev/null
    )

    return 1
}

detect_panel() {
    info "Mendeteksi instalasi Pterodactyl..."

    if ! find_panel; then
        die "Instalasi Pterodactyl tidak ditemukan."
    fi

    success "Panel ditemukan: $PANEL_DIR"
}

# ============================================================
# VERSION
# ============================================================

detect_version() {
    PANEL_VERSION="Unknown"

    if [[ -f "$PANEL_DIR/artisan" ]] && command_exists php; then
        PANEL_VERSION="$(
            cd "$PANEL_DIR" &&
            php artisan --version 2>/dev/null |
            grep -oE '[0-9]+\.[0-9]+\.[0-9]+' |
            head -n1
        )"
    fi

    [[ -z "$PANEL_VERSION" ]] && PANEL_VERSION="Unknown"

    echo -e "Version : ${CYAN}${PANEL_VERSION}${RESET}"

    case "$PANEL_VERSION" in
        1.6.*)
            success "Pterodactyl 1.6.x terdeteksi."
            ;;
        1.7.*)
            success "Pterodactyl 1.7.x terdeteksi."
            ;;
        *)
            warning "Versi tidak terdeteksi sebagai 1.6.x/1.7.x."
            warning "Melanjutkan menggunakan mode kompatibilitas."
            ;;
    esac
}

# ============================================================
# BACKUP
# ============================================================

create_backup() {
    local date
    date="$(date '+%Y%m%d-%H%M%S')"

    BACKUP_DIR="/var/www/pterodactyl-backup-${date}"

    info "Membuat backup..."

    mkdir -p "$BACKUP_DIR" || \
        die "Tidak dapat membuat folder backup."

    cp -a "$PANEL_DIR/resources" "$BACKUP_DIR/" || {
        rm -rf "$BACKUP_DIR"
        die "Backup resources gagal."
    }

    [[ -f "$PANEL_DIR/package.json" ]] &&
        cp -a "$PANEL_DIR/package.json" "$BACKUP_DIR/"

    [[ -f "$PANEL_DIR/yarn.lock" ]] &&
        cp -a "$PANEL_DIR/yarn.lock" "$BACKUP_DIR/"

    [[ -f "$PANEL_DIR/package-lock.json" ]] &&
        cp -a "$PANEL_DIR/package-lock.json" "$BACKUP_DIR/"

    cat > "$BACKUP_DIR/backup-info.txt" <<EOF
SISI STORE Pterodactyl Theme Backup

Date: $(date)
Panel: $PANEL_DIR
Version: $PANEL_VERSION
Theme URL: $THEME_URL
EOF

    success "Backup dibuat:"
    echo -e "        ${CYAN}${BACKUP_DIR}${RESET}"
}

# ============================================================
# DOWNLOAD
# ============================================================

download_theme() {
    TEMP_DIR="$(mktemp -d /tmp/sisi-theme.XXXXXX)"

    THEME_FILE="$TEMP_DIR/theme"

    info "Download tema..."

    curl \
        --fail \
        --location \
        --show-error \
        --progress-bar \
        "$THEME_URL" \
        -o "$THEME_FILE" || {
            die "Gagal mendownload tema."
        }

    if [[ ! -s "$THEME_FILE" ]]; then
        die "File tema kosong."
    fi

    success "Tema berhasil didownload."
}

# ============================================================
# EXTRACT
# ============================================================

extract_theme() {
    THEME_EXTRACT="$TEMP_DIR/extracted"

    mkdir -p "$THEME_EXTRACT"

    info "Extract tema..."

    local file_type
    file_type="$(file -b "$THEME_FILE" 2>/dev/null || true)"

    if [[ "$file_type" == *"Zip archive"* ]]; then

        unzip -q "$THEME_FILE" -d "$THEME_EXTRACT" || \
            die "Extract ZIP gagal."

    elif [[ "$file_type" == *"gzip compressed"* ]]; then

        tar -xzf "$THEME_FILE" -C "$THEME_EXTRACT" || \
            die "Extract TAR.GZ gagal."

    elif [[ "$file_type" == *"tar archive"* ]]; then

        tar -xf "$THEME_FILE" -C "$THEME_EXTRACT" || \
            die "Extract TAR gagal."

    else
        die "Format tema tidak dikenali. Gunakan ZIP atau TAR.GZ."
    fi

    success "Tema berhasil diekstrak."
}

# ============================================================
# FIND THEME STRUCTURE
# ============================================================

find_theme_resources() {

    if [[ -d "$THEME_EXTRACT/resources" ]]; then
        THEME_RESOURCES="$THEME_EXTRACT/resources"
        return 0
    fi

    local found
    found="$(
        find "$THEME_EXTRACT" \
            -type d \
            -name resources \
            -print \
            -quit \
            2>/dev/null
    )"

    if [[ -n "$found" ]]; then
        THEME_RESOURCES="$found"
        return 0
    fi

    if [[ -d "$THEME_EXTRACT/views" ||
          -d "$THEME_EXTRACT/scripts" ||
          -d "$THEME_EXTRACT/js" ||
          -d "$THEME_EXTRACT/css" ]]; then

        THEME_RESOURCES="$THEME_EXTRACT"
        return 0
    fi

    return 1
}

# ============================================================
# INSTALL
# ============================================================

install_theme() {
    info "Memasang file tema..."

    if ! find_theme_resources; then
        die "Struktur tema tidak valid."
    fi

    cp -a "$THEME_RESOURCES/." "$PANEL_DIR/resources/" || \
        die "Gagal menyalin file tema."

    success "File tema berhasil dipasang."
}

# ============================================================
# BUILD
# ============================================================

build_panel() {
    cd "$PANEL_DIR" || \
        die "Tidak dapat masuk ke direktori panel."

    if ! command_exists php; then
        die "PHP tidak ditemukan."
    fi

    info "Mengaktifkan maintenance mode..."

    php artisan down >/dev/null 2>&1 || \
        warning "Maintenance mode gagal diaktifkan."

    local build_ok=false

    echo
    info "Mendeteksi build system..."

    if [[ -f "yarn.lock" ]] && command_exists yarn; then

        echo -e "Build   : ${CYAN}yarn build:production${RESET}"
        echo

        if yarn build:production; then
            build_ok=true
        fi

    elif [[ -f "package-lock.json" ]] && command_exists npm; then

        echo -e "Build   : ${CYAN}npm run build${RESET}"
        echo

        if npm run build; then
            build_ok=true
        fi

    elif command_exists yarn; then

        echo -e "Build   : ${CYAN}yarn build:production${RESET}"
        echo

        if yarn build:production; then
            build_ok=true
        fi

    elif command_exists npm; then

        echo -e "Build   : ${CYAN}npm run build${RESET}"
        echo

        if npm run build; then
            build_ok=true
        fi

    else
        php artisan up >/dev/null 2>&1 || true
        die "Yarn/npm tidak ditemukan."
    fi

    if [[ "$build_ok" != true ]]; then
        error "Build panel gagal."
        warning "Maintenance mode dibiarkan aktif untuk keamanan."
        warning "Gunakan backup berikut untuk restore:"
        echo -e "        ${CYAN}${BACKUP_DIR}${RESET}"
        exit 1
    fi

    success "Build selesai."

    php artisan optimize:clear >/dev/null 2>&1 || true

    info "Menonaktifkan maintenance mode..."

    php artisan up >/dev/null 2>&1 || \
        die "Gagal mengaktifkan kembali panel."

    success "Panel kembali online."
}

# ============================================================
# PERMISSION
# ============================================================

fix_permissions() {
    info "Memperbaiki permission..."

    chown -R www-data:www-data "$PANEL_DIR" || \
        die "Gagal memperbaiki ownership."

    success "Permission selesai."
}

# ============================================================
# RESTORE
# ============================================================

restore_backup() {

    echo
    echo -e "${NEON}SISI STORE - RESTORE BACKUP${RESET}"
    echo

    mapfile -t backups < <(
        find /var/www \
            -maxdepth 1 \
            -type d \
            -name 'pterodactyl-backup-*' \
            -print 2>/dev/null |
        sort -r
    )

    if [[ "${#backups[@]}" -eq 0 ]]; then
        warning "Backup tidak ditemukan."
        return
    fi

    local i=1

    for backup in "${backups[@]}"; do
        echo "[$i] $(basename "$backup")"
        ((i++))
    done

    echo
    read -rp "Pilih backup [1-${#backups[@]}]: " choice

    if ! [[ "$choice" =~ ^[0-9]+$ ]]; then
        error "Pilihan tidak valid."
        return
    fi

    if (( choice < 1 || choice > ${#backups[@]} )); then
        error "Backup tidak tersedia."
        return
    fi

    local selected="${backups[$((choice-1))]}"

    echo
    warning "Resources panel saat ini akan diganti dengan backup."
    read -rp "Lanjutkan? [y/N]: " confirm

    if [[ ! "$confirm" =~ ^[Yy]$ ]]; then
        info "Restore dibatalkan."
        return
    fi

    cd "$PANEL_DIR" || return

    php artisan down >/dev/null 2>&1 || true

    if [[ ! -d "$selected/resources" ]]; then
        php artisan up >/dev/null 2>&1 || true
        error "resources tidak ditemukan dalam backup."
        return
    fi

    rm -rf "$PANEL_DIR/resources"

    cp -a "$selected/resources" "$PANEL_DIR/" || {
        php artisan up >/dev/null 2>&1 || true
        error "Restore gagal."
        return
    }

    fix_permissions

    if [[ -f "yarn.lock" ]] && command_exists yarn; then
        yarn build:production || warning "Build setelah restore gagal."

    elif command_exists npm; then
        npm run build || warning "Build setelah restore gagal."
    fi

    php artisan optimize:clear >/dev/null 2>&1 || true
    php artisan up >/dev/null 2>&1 || true

    success "Backup berhasil direstore."
}

# ============================================================
# INSTALL PROCESS
# ============================================================

install_process() {

    echo -e "${PURPLE}"
    echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
    echo -e "${WHITE}          SISI STORE INSTALLER${RESET}"
    echo -e "${PURPLE}━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━${RESET}"
    echo

    check_root
    detect_os
    install_dependencies
    detect_panel
    detect_version

    echo
    echo -e "Panel : ${CYAN}${PANEL_DIR}${RESET}"
    echo -e "Theme : ${CYAN}${THEME_URL}${RESET}"
    echo

    read -rp "Lanjutkan instalasi? [y/N]: " confirm

    if [[ ! "$confirm" =~ ^[Yy]$ ]]; then
        info "Instalasi dibatalkan."
        exit 0
    fi

    echo

    create_backup
    download_theme
    extract_theme
    install_theme
    build_panel
    fix_permissions

    echo
    echo -e "${GREEN}"
    cat <<'EOF'
╔══════════════════════════════════════════════╗
║                                              ║
║          ✓ INSTALLASI BERHASIL              ║
║                                              ║
╚══════════════════════════════════════════════╝
EOF
    echo -e "${RESET}"

    echo -e "Backup tersimpan di:"
    echo -e "${CYAN}${BACKUP_DIR}${RESET}"
    echo
}

# ============================================================
# MAIN MENU
# ============================================================

main() {

    show_logo
    check_root

    echo -e "${WHITE}[1] Install / Update Theme${RESET}"
    echo -e "${WHITE}[2] Restore / Uninstall Theme${RESET}"
    echo -e "${WHITE}[3] Exit${RESET}"
    echo

    read -rp "Pilih [1-3]: " option

    case "$option" in

        1)
            install_process
            ;;

        2)
            detect_panel
            restore_backup
            ;;

        3)
            exit 0
            ;;

        *)
            error "Pilihan tidak valid."
            exit 1
            ;;

    esac
}

main
