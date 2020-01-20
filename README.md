# Cách Build SailfishOS #
## Bắt Đầu
Tước Hết Bạn Cần Tạo Một User Khác / Cái Này Tùy Ý ( Bỏ Qua Cũng Đc )
```bash
   sudo useradd build -s /bin/bash -m -G wheel -c "SFOS Builder"
   sudo passwd build
   su - build
```
## Bước 1:
Bạn Cần add Những Thứ Này Vào ~/.bashrc
```bash
   # SailfishOS
   export HISTFILE="$HOME/.bash_history"
   export HISTSIZE=1000
   export HISTCONTROL=ignoreboth
   export PATH=$HOME/bin:$PATH
   export PLATFORM_SDK_ROOT="/srv/mer"
   export ANDROID_ROOT="$HOME/Sailfish/src"
   shopt -s histappend
   alias sfossdk="$PLATFORM_SDK_ROOT/sdks/sfossdk/mer-sdk-chroot"
   alias sfos_sdk="sfossdk"
   alias platform_sdk="sfossdk"
   alias plat_sdk="sfossdk"
   alias platformsdk="sfossdk"
   alias platsdk="sfossdk"
```
Tiếp Theo Bạn Cần Tạo File .hadk.env ở ~/ và dán nội dung này vào :
```bash
   export MER_ROOT="/srv/mer"
   export PLATFORM_SDK_ROOT="/"
   export ANDROID_ROOT="$HOME/Sailfish/src"
   export VENDOR="xiaomi"
   export DEVICE="whyred"
   export PORT_ARCH="armv7hl"

   # If not running interactively, don't do anything else
   [[ $- != *i* ]] && return

   echo "$ export LANG=C LC_ALL=POSIX"
   export LANG=C LC_ALL=POSIX
   echo "$ cd \$ANDROID_ROOT"
   cd $ANDROID_ROOT
```
Tiếp Theo Là File .mersdk.profile
```bashrc
builder_script="rpm/dhd/helpers/build_packages.sh"
[ -d /etc/bash_completion.d ] && for i in /etc/bash_completion.d/*; do . $i; done
export PS1="PLATFORM_SDK $PS1"
export HISTFILE="$HOME/.bash_history-sfossdk"
export RELEASE=`cat /etc/os-release | grep VERSION_ID | cut -d"=" -f2`

alias host="exit"
alias ha_build="ubu-chroot -r $PLATFORM_SDK_ROOT/sdks/ubuntu"
alias habuild="ha_build"
alias build_all_pkgs="build_all_packages"
alias build_all="build_all_packages"
alias bp="build_packages"
alias build_pkgs="build_packages"
alias build_droid_hal="build_packages -d"
alias build_hal="build_droid_hal"
alias build_device_configs="build_packages -c"
alias build_configs="build_device_configs"
alias build_cfgs="build_device_configs"
alias do_mic_build="run_mic_build"
alias mic_build="run_mic_build"
alias build_sfos="run_mic_build"
alias build_sailfish="run_mic_build"
alias build_img="run_mic_build"
alias build_image="run_mic_build"
alias reset_droid_repos="sudo rm -rf $ANDROID_ROOT/rpm/ $ANDROID_ROOT/hybris/droid-{configs,hal-version-$DEVICE}* $ANDROID_ROOT/droid-hal-$DEVICE.log $ANDROID_ROOT/.last_device; unset last_device; choose_target"
alias reset_repos="reset_droid_repos"
alias choose_device="choose_target"
alias switch_target="choose_target"
alias switch_device="choose_target"

hadk() {
	echo
	source $HOME/.hadk.env
	echo "Env setup for $DEVICE"
}

clone_src() {
	path="$ANDROID_ROOT/$3/"
	mkdir -p "$path"
	git clone --recurse -b $2 https://github.com/SailfishOS-Whyred-sdm660/$1 "$path" &> /dev/null
}

update_src() {
	path="$ANDROID_ROOT/$1/"
	[ ! -d "$path" ] && exit 1
	# TODO: Fix updating properly; force all unless local changes detected?
	cd "$path" && git fetch &> /dev/null && git pull --recurse-submodules &> /dev/null
}

choose_target() {
	echo -e "\nWhich hybris-15.1 device would you like to build for?"
	echo -e "\n  1. whyred (Xiaomi Redmi Note 5 Pro)"
	echo -e "  2. dumpling     (OnePlus 5T)\n"
	read -p "Choice: (1/2) " target

	# Setup variables
	device="whyred"
	[ "$target" = "2" ] && device="dumpling"
	branch="hybris-15.1"
	[ "$device" = "dumpling" ] && branch="dumpling-15.1"
	[ -f "$ANDROID_ROOT/.last_device" ] && last_device="$(<$ANDROID_ROOT/.last_device)"

	if [ "$device" != "$last_device" ]; then
		if [ ! -z "$last_device" ]; then
			echo "WARNING: All current changes in SFOS local droid repos WILL be discarded if you continue!"
			read -p "Would you like to continue? (y/N) " ans
			ans=`echo "$ans" | xargs | tr "[y]" "[Y]"`
			if [ "$ans" != "Y" ]; then
				hadk
				return 1
			fi

			echo "Discarded local droid HAL & configs for $last_device!"
			rm -rf $ANDROID_ROOT/rpm* $ANDROID_ROOT/hybris/droid-{configs,hal-version-}*
		fi

		printf "Cloning droid HAL & configs for $device..."
		clone_src "droid-hal-whyred" "$branch" "rpm" &&
		clone_src "droid-config-whyred" "$branch" "hybris/droid-configs" &&
		clone_src "droid-hal-version-whyred" "$branch" "hybris/droid-hal-version-$device"
		(( $? == 0 )) && echo " done!" || echo " fail! exit code: $?"

		echo "$device" > "$ANDROID_ROOT/.last_device"
	else
		printf "Updating droid HAL & configs for $device..."
		update_src "rpm" &&
		update_src "hybris/droid-configs" &&
		update_src "hybris/droid-hal-version-$device"
		(( $? == 0 )) && echo " done!" || echo " fail! exit code: $?"
	fi

	sed "s/DEVICE=.*/DEVICE=\"$device\"/" -i $HOME/.hadk.env
	hadk
}

build_all_packages() {
	cd $ANDROID_ROOT

	echo "$ $builder_script $@"
	$builder_script $@
}

build_packages() {
	cd $ANDROID_ROOT

	if (( $# == 0 )); then
		build_packages -c
		return
	fi

	echo "$ $builder_script $@"
	$builder_script $@ || return
}

run_mic_build() {
	if [ -z $UPDATES_CHECKED ]; then
		# Function to compare version strings
		vercomp() {
			[ "$1" = "$2" ] && return 0 # =
			[ "$1" = `printf "$1\n$2" | sort -t '.' -k 1,1 -k 2,2 -k 3,3 -k 4,4 -g | head -1` ] && return 1 # <
			return 2 # >
		}

		# Fetch latest public release version
		local tmp=`curl -s https://en.wikipedia.org/wiki/Sailfish_OS | grep -B 2 "^<td>Public release$" | grep "^<td>v" | tail -1` # e.g. "<td>v3.0.3.10</td>"
		tmp=${tmp#*v} # e.g. "3.0.3.10</td>"
		local LATEST_RELEASE=${tmp::${#tmp}-5} # e.g. "3.0.3.10"
		local LATEST_TOOLING=`echo $LATEST_RELEASE | cut --complement -d"." -f4-` # e.g. "3.0.3"
		local CURRENT_TOOLING=`echo $RELEASE | cut --complement -d"." -f4-` # e.g. "3.1.0"

		# Can we build latest w/ current tooling (e.g. '3.0.3' vs '3.1.0')
		vercomp "$LATEST_TOOLING" "$CURRENT_TOOLING"
		local res=$?
		if (( $res == 0 )); then
			# TODO Check if installed tooling is latest available from http://releases.sailfishos.org/sdk/targets/

			# Only use "latest version" if it's actually newer
			vercomp "$LATEST_RELEASE" "$RELEASE"
			res=$?
			if (( $res == 2 )); then
				RELEASE="$LATEST_RELEASE"
				echo ">> Targeting latest public release $RELEASE."

			# Out-of-date version history => Use local tooling
			else
				echo ">> Targeting installed tooling release $RELEASE."
			fi

		# Out-of-date version history => Use local tooling
		elif (( $tmp == 1 )); then
			echo ">> Targeting installed tooling release $RELEASE."

		# Can't build w/ current tooling => Check if tooling updates available
		else
			curl -s http://releases.sailfishos.org/sdk/targets/ | fgrep $LATEST_TOOLING &>/dev/null && echo ">> Build target updates available ($CURRENT_TOOLING.x => $LATEST_TOOLING.x)! Resources: http://releases.sailfishos.org/sdk/targets/ https://git.io/fjM1D"
			echo ">> Currently targeting installed tooling release $RELEASE."
		fi

		UPDATES_CHECKED=1
	fi

	# TODO: Add support to build image using OBS KS (devel/testing), do local fixups etc
	build_packages -i
}

choose_target
```
Tiếp Theo Là File .mersdkubu.profile :
```bashrc

hadk() { source $HOME/.hadk.env; echo "Env setup for $DEVICE"; }
export PS1="HABUILD_SDK [\${DEVICE}] $PS1"
hadk

sdk_prompt() { echo "$1: enter PLATFORM_SDK first by pressing CTRL + D & try again!"; }
alias zypper="sdk_prompt zypper"
alias sb2="sdk_prompt sb2"
alias sfossdk="exit"
alias sfos_sdk="exit"
alias platform_sdk="exit"
alias plat_sdk="exit"
alias platformsdk="exit"
alias platsdk="exit"

export HISTFILE="$HOME/.bash_history-habuild"

if [ -f build/envsetup.sh ]; then
	echo "$ source build/envsetup.sh"
	source build/envsetup.sh
	echo "$ breakfast $DEVICE"
	breakfast $DEVICE
	echo "$ export USE_CCACHE=1"
	export USE_CCACHE=1
fi
```
