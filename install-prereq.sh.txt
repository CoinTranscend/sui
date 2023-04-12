#!/bin/bash

# Check distribution type
if [ -f /etc/debian_version ]; then
    PACKAGE_MANAGER="apt-get"
elif [ -f /etc/redhat-release ] || [ -f /etc/system-release ] || grep -q "Amazon Linux" /etc/os-release; then
    PACKAGE_MANAGER="yum"
else
    echo "Unsupported distribution. Exiting."
    exit 1
fi

# Update package list
echo "Updating package list..."
sudo $PACKAGE_MANAGER update -y

# Function to check and replace curl if necessary
check_and_replace_curl() {
    existing_version=$(curl --version 2>/dev/null | head -n 1 | awk '{print $2}')
    if [ -z "$existing_version" ]; then
        echo "Installing/updating curl..."
        sudo $PACKAGE_MANAGER install -y curl
        return
    fi

    latest_version=$(sudo $PACKAGE_MANAGER info curl | grep -oP 'Version\s*:\s*\K[0-9.]+')
    if [ "$existing_version" != "$latest_version" ]; then
        echo "Existing curl version: $existing_version"
        echo "Latest curl version: $latest_version"
        read -p "Do you want to replace the existing version with the latest version? (y/n): " yn
        case $yn in
            [Yy]* ) sudo $PACKAGE_MANAGER remove -y curl-minimal 2>/dev/null || true
                    sudo $PACKAGE_MANAGER install -y curl;;
            * ) echo "Skipping curl update.";;
        esac
    else
        echo "curl is already up to date."
    fi
}

# Install packages and print their status
declare -A packages
if [ "$PACKAGE_MANAGER" == "apt-get" ]; then
    packages=( ["git-all"]="git" ["cmake"]="cmake" ["gcc"]="gcc" ["libssl-dev"]="libssl" ["pkg-config"]="pkg-config" ["libclang-dev"]="libclang" ["libpq-dev"]="libpq" ["build-essential"]="build-essential")
elif [ "$PACKAGE_MANAGER" == "yum" ]; then
    packages=( ["git-all"]="git" ["cmake"]="cmake" ["gcc"]="gcc" ["openssl-devel"]="libssl" ["pkgconfig"]="pkg-config" ["clang-devel"]="libclang" ["postgresql-devel"]="libpq" ["make"]="build-essential")
fi

# Check and replace curl if necessary
check_and_replace_curl

for package in "${!packages[@]}"; do
    echo "Installing/updating ${packages[$package]}..."
    sudo $PACKAGE_MANAGER install -y $package
done

# Install Rust
echo "Installing/updating Rust..."
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh -s -- -y

# Update Rust stable
echo "Sourcing Rust environment variables..."
source $HOME/.cargo/env

echo "Updating Rust stable version..."
rustup update stable

echo "All packages installed/updated successfully!"
