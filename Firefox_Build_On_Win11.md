#### 系统要求
```
Memory: 4GB RAM minimum, 8GB+ recommended.
Disk Space: At least 40GB of free disk space.
Win11 or Win10 with lastest update
```
#### 安装Visual [Studio Build Tools](https://visualstudio.microsoft.com/downloads/#build-tools-for-visual-studio-2022)组件
```
Desktop development with C++

MSVC v143 - VS 2022 C++ x64/x86 build tools
Windows 10 SDK (at least 10.0.19041.0)		# 对应操作系统的最新SDK
C++ ATL for v143 build tools (x86 and x64)
```
#### 安装[MozillaBuild](https://ftp.mozilla.org/pub/mozilla/libraries/win32/MozillaBuildSetup-Latest.exe)到D:\work
修改`start-shell.bat`，添加环境变量`PATH LIB INCLUDE`
```
REM Reset some env vars.
SET MOZBUILD_STATE_PATH=/d/work/.mozbuild
SET PATH=D:\work\.mozbuild\cbindgen;D:\work\.mozbuild\dump_syms;D:\work\.mozbuild\fix-stacks;D:\work\.mozbuild\git-cinnabar;D:\work\.mozbuild\minidump-stackwalk;D:\work\.mozbuild\mozmake;D:\work\.mozbuild\nasm;D:\work\.mozbuild\node;D:\work\.mozbuild\nsis;D:\work\.mozbuild\sccache;D:\work\.mozbuild\winchecksec;D:\work\PortableGit64;D:\work\PortableGit64\bin;C:\Users\Administrator\.mozbuild\git-cinnabar;%PATH%
SET CYGWIN=D:\work\cygwin64\bin
SET INCLUDE=D:\work\.mozbuild\clang\include;%INCLUDE%
SET LIB=D:\work\.mozbuild\clang\lib;D:\work\.mozbuild\clang\lib\clang\15.0.5\lib\windows;%LIB%
SET GITDIR=
SET MOZILLABUILD=%~dp0

SET VC_REDISTDIR=C:\Program Files (x86)\Microsoft Visual Studio\2022\BuildTools\VC\Redist\MSVC\14.34.31931
SET WIN32_REDIST_DIR=%VC_REDISTDIR%\x64\Microsoft.VC143.CRT

SET UCRT_REDISTDIR=C:\Program Files (x86)\Windows Kits\10
SET WIN_UCRT_REDIST_DIR=%UCRT_REDISTDIR%\Redist\ucrt\DLLs\x64

SET WINDOWSSDKDIR=C:\Program Files (x86)\Windows Kits\10
```
#### 安装 `Build Tools for Visual Studio 2022`
#### 安装绿色版版`git`到`D:\work\PortableGit64`
#### 安装`Cygwin`到`D:\work\cygwin64`（`Cygwin`非必须）

#### 启动`start-shell.bat`通过`git`拉取源码（目录为 `D:\work\mozilla-unified`）
```
mkdir /d/work
cd /d/work
wget https://hg.mozilla.org/mozilla-central/raw-file/default/python/mozboot/bin/bootstrap.py
python3 bootstrap.py --vcs=git
```
  源码获取完成后会自动安装`bootstrap toolchain`到`MOZBUILD_STATE_PATH` 目录`D:\work\.mozbuild`

`cd mozilla-unified`
`git branch -v`或者`git branch -a`查看分支  
```
mozilla-central: 最新的开发版(默认分支)
mozilla-beta: 测试版分支
mozilla-release: 正式版分支
```

#### 配置`.mozconfig`配置文件后开始编译（如设置编译目录为`D:\work\obju64-release`）
### PGO优化：
#### 生成首次编译配置
```
cd D:/work/obju64-release	或 cd /d/work/obju64-release
../mozilla-release/configure --enable-profile-generate=cross
```
#### 开始编译
`mozmake -j5`
#### 打包
`mozmake package`

#### 生成PGO优化文件
`MOZ_HEADLESS=1 DISPLAY=22 JARLOG_FILE=jarlog/en-US.log _virtualenvs/build/Scripts/python ../mozilla-release/build/pgo/profileserver.py`

`mozmake maybe_clobber_profiledbuild`

  ##### 或未修改`mozilla-unified\Make.in`情况下，执行
  
	 `$(LLVM_PROFDATA) show --value-cutoff=1024 --all-functions $(DEPTH)/instrumented/merged.profdata --output $(DEPTH)/orderfile.txt`

#### 生成二次编译配置并编译
```../mozilla-release/configure --enable-profile-use=cross --enable-lto=cross
mozmake -j5
mozmake package
```
#### 生成中文安装包，（目录为`obju64-release/dist/installers/sea`）
`mozmake installers-zh-CN`


## 源码修改及优化
### 通过`maybe_clobber_profiledbuild`生成PGO优化相关文件
#### 修改`mozilla-unified\Make.in`
```
...
### 添加以下
.PHONY: package-generated-sources
package-generated-sources:
	$(call py_action,package_generated_sources,'$(DIST)/$(PKG_PATH)$(GENERATED_SOURCE_FILE_PACKAGE)')

# PGO support, but we can't do this test in client.mk
# No point in clobbering if PGO has been explicitly disabled.
ifdef NO_PROFILE_GUIDED_OPTIMIZE
maybe_clobber_profiledbuild: clean
else
maybe_clobber_profiledbuild:
ifneq (,$(findstring clang,$(CC_TYPE)))
	-rm -rf $(DEPTH)/instrumented
	-mkdir $(DEPTH)/instrumented
	$(LLVM_PROFDATA) merge -o $(DEPTH)/instrumented/merged.profdata $(DEPTH)/*.profraw
ifeq ($(OS_ARCH),WINNT)	
	$(LLVM_PROFDATA) show --value-cutoff=1024 --all-functions $(DEPTH)/instrumented/merged.profdata --output $(DEPTH)/orderfile.txt
endif # !WINNT
endif # !clang
endif # NO_PROFILE_GUIDED_OPTIMIZE

.PHONY: maybe_clobber_profiledbuild

### 添加以上
ifdef JS_STANDALONE
...
```

#### 修改mozilla-unified\browser\app\profile\firefox.js
```
不显示赞助商项目
    pref("browser.topsites.contile.enabled", false);        about:newtab
    pref("services.sync.prefs.sync.browser.newtabpage.activity-stream.showSponsored", false);
    pref("services.sync.prefs.sync.browser.newtabpage.activity-stream.showSponsoredTopSites", false);
    pref("browser.newtabpage.activity-stream.discoverystream.spocs.personalized", false);
    pref("browser.urlbar.suggest.quicksuggest.sponsored", false, sticky);
    pref("browser.newtabpage.activity-stream.showSponsoredTopSites", false);           about:newtab
    //pref("browser.topsites.contile.endpoint", "https://contile.services.mozilla.com/v1/tiles");
    pref("browser.urlbar.suggest.quicksuggest.nonsponsored", true, sticky);

标签栏不显示tabmanager
    pref("browser.tabs.tabmanager.enabled", false);

// smoothScroll scrolling.
// scrolling
pref("mousewheel.withnokey.sysnumlines",false);
pref("mousewheel.withnokey.numlines", 7);
pref("mousewheel.min_line_scroll_amount", 50);
pref("general.smoothScroll.durationToIntervalRatio", 670);
pref("general.smoothScroll.mouseWheel.durationMaxMS", 420);
pref("mousewheel.acceleration.factor", 10);
pref("mousewheel.acceleration.start", 3);
pref("mousewheel.default.delta_multiplier_y", 180);

// 非签名语言包
    pref("extensions.langpacks.signatures.required", false);

禁用pocket
    pref("extensions.pocket.enabled", false);

禁用隐私收集
    pref("datareporting.healthreport.uploadEnabled", false);
    pref("browser.discovery.enabled", false);
    pref("app.shield.optoutstudies.enabled", false);
    // Discovery prefs
    pref("browser.discovery.enabled", false);
    pref("browser.discovery.containers.enabled", false);

不显示Firefox-view
    pref("browser.tabs.firefox-view", false);

禁用自动升级
      pref("app.update.auto", false);
```
### 启用libportable
#### 修改`mozilla-unified\toolkit\library\nsDllMain.cpp`
```
 #include "mozilla/Assertions.h"
 #include "mozilla/WindowsVersion.h"
 
### 添加以下
#if defined(_MSC_VER) && defined(TT_MEMUTIL)
#if defined(_M_IX86)
#pragma comment(lib,"portable32.lib")
#pragma comment(linker, "/include:_GetCpuFeature_tt")
#elif defined(_M_AMD64) || defined(_M_X64)
#pragma comment(lib,"portable64.lib")
#pragma comment(linker, "/include:GetCpuFeature_tt")
#endif
#endif /* _MSC_VER && TT_MEMUTIL */
### 添加以上

 #if defined(__GNUC__)
 // If DllMain gets name mangled, it won't be seen.
 extern "C" {
```
#### 修改`mozilla-unified\browser\installer\package-manifest.in`
```
...
#ifdef XP_WIN
#if MOZ_PACKAGE_MSVC_DLLS
@BINPATH@/@MSVC_C_RUNTIME_DLL@
@BINPATH@/@MSVC_CXX_RUNTIME_DLL@

###添加以下
@BINPATH@/portable*.dll
@BINPATH@/portable.ini
###添加以上
#endif
...
```
#### 添加libportable文件
##### CMD编译libportable
```
# 启动CMD后执行
call "C:\Program Files (x86)\Microsoft Visual Studio\2022\BuildTools\VC\Auxiliary\Build\vcvars64.bat"
set LIBPORTABLE_PATH=D:\work\.mozbuild\clang  # 如不执行，则需下一步 安装libportable

cd /d D:\work\libportable
nmake -f Makefile.msvc clean
nmake -f Makefile.msvc CC=clang-cl  # 编译命令，可略过，执行 nmake -f Makefile.msvc CC=clang-cl install
nmake -f Makefile.msvc CC=clang-cl install  # 如不执行，则需下一步 安装libportable
```
##### 安装libportable
```
libportable 的头文件 portable.h  > D:\work\.mozbuild\clang\include
将 编译libportable 生成的 portable64.lib > D:\work\.mozbuild\clang\lib
将 编译libportable 生成的 portable64.dll、portable.ini > D:\work\.mozbuild\obju64-release\dist_source\bin
```

#### 修改`zh-CN`语言包源码
##### 语言包源码
`https://hg.mozilla.org/l10n-central/zh-CN`
默认语言包源码目录为`MOZBUILD_STATE_PATH`
或在.mozconfig中指定(如 d:/work/l10n-central\zh-CN)
`ac_add_options --with-l10n-base=d:/work/l10n-central`
##### 修改`l10n-central\zh-CN\browser\chrome\browser\browser.properties`
```
browser.startup.homepage = about:newtab	  # 设置Firefox启动主页为新标签页
```

#### `.mozconfig`配置文件
```
# This file specifies the build flags for Iceweasel.  You can use it by adding:
# to the top of your mozconfig file.
# PGO build (mach build -v 2>&1 | tee -i ../build_x86.log)
. $topsrcdir/browser/config/mozconfig
ac_add_options --with-app-name=Firefox

mk_add_options MOZ_OBJDIR=../obju64-release
mk_add_options MOZ_MAKE_FLAGS=-j5

# for 64-bit build
ac_add_options --host=x86_64-pc-mingw32
ac_add_options --target=x86_64-pc-mingw32

# not need mozbuild install by default dir
#export MOZBUILD_STATE_PATH=/d/work/mozbuild
#export MOZILLABUILD=/d/work/mozbuild

# need when mozillaBuild
#export WIN32_REDIST_DIR="/c/Program Files (x86)/Microsoft Visual Studio/2022/BuildTools/VC/Redist/MSVC/14.34.31931/x64/Microsoft.VC143.CRT"
#export WINDOWSSDKDIR="/c/Program Files (x86)/Windows Kits/10"
#export WIN_UCRT_REDIST_DIR="$WINDOWSSDKDIR/redist/10.0.22000.0/ucrt/dlls/x64"

#export LIB="$LIB;D:\\src\\llvm8\\lib\\clang\\8.0.0\\lib\\windows"
#export LIB="$LIB;/C/Users/Administrator/.mozbuild/clang/lib;/C/Users/Administrator/.mozbuild/clang/lib/clang/15.0.5/lib/windows"
#

# optimize
ac_add_options MOZ_PGO=1
export CC=clang-cl
export CXX=clang-cl
export LINKER=lld-link
export RUSTC_OPT_LEVEL=2
#export ENABLE_CLANG_PLUGIN=1
#export MOZ_LTO=cross

ac_add_options --enable-optimize="-O2 -DTT_MEMUTIL -clang:-ftree-vectorize -clang:-march=skylake -clang:-mtune=skylake"
export RUSTFLAGS="$RUSTFLAGS -Ctarget-cpu=skylake"
#ac_add_options --enable-optimize="-O2 -DTT_MEMUTIL"
ac_add_options --enable-rust-simd
ac_add_options --enable-jemalloc
ac_add_options --enable-install-strip
ac_add_options --enable-strip
# enable clang plugins
# ac_add_options --enable-mozsearch-plugin
ac_add_options --enable-clang-plugin

# 1.st
#ac_add_options --enable-profile-generate=cross
# 2.st
#ac_add_options --enable-profile-use=cross
#ac_add_options --enable-lto=cross

# disable things
ac_add_options --disable-tests
ac_add_options --disable-debug
ac_add_options --disable-updater
ac_add_options --disable-crashreporter
ac_add_options --disable-parental-controls
ac_add_options --disable-necko-wifi
#ac_add_options --disable-webrtc
# Encrypted Media Extensions 
#ac_add_options --disable-eme
ac_add_options --disable-sandbox
ac_add_options --disable-debug
ac_add_options --disable-debug-symbols
ac_add_options --disable-maintenance-service
ac_add_options --disable-accessibility
ac_add_options --disable-webspeech
ac_add_options --disable-webspeechtestbackend
ac_add_options --disable-negotiateauth
ac_add_options --disable-system-policies
ac_add_options --disable-geckodriver
ac_add_options --disable-dmd
ac_add_options --disable-profiling
MOZ_TELEMETRY_REPORTING=

# Enable
ac_add_options --allow-addon-sideload
# LEGACY_EXTENSIONS
ac_add_options "MOZ_ALLOW_LEGACY_EXTENSIONS=1"
# Add-on signing is not required
MOZ_REQUIRE_SIGNING=

# Branding
export MOZILLA_OFFICIAL=1
ac_add_options --enable-update-channel=release
ac_add_options --without-wasm-sandboxed-libraries
ac_add_options --with-branding=browser/branding/official

ac_add_options --with-l10n-base=d:/work/l10n-central
#ac_add_options --enable-ui-locale=zh-CN

# crt dir need by fx_build.sh
#WIN32_REDIST_DIR=${VC_REDISTDIR%?}\\x64\\Microsoft.VC143.CRT
#WIN_UCRT_REDIST_DIR=${UCRT_REDISTDIR%?}\\Redist\\ucrt\\DLLs\\x64
```

#### `MOZBUILD_STATE_PATH` 内编译相关工具链
```
cbindgen
clang
mozmake
node
nasm
nsis

# 以下非编译必须，./mach bootstrap相关
git-cinnabar	# git获取源码修改
clang-tools
sccache
sysroot-wasm32-wasi
fix-stacks
glean
dump_syms
minidump-stackwalk
srcdirs
winchecksec
```

##### 文档及第三方资源
- [win编译](https://firefox-source-docs.mozilla.org/setup/windows_build.html)
- [语言包源码](https://firefox-source-docs.mozilla.org/build/buildsystem/locales.html)
- [源码分支](https://firefox-source-docs.mozilla.org/contributing/vcs/mercurial.html#mozilla-release)
- [libportable](https://github.com/adonais/libportable)
- [Iceweasel](https://github.com/adonais/iceweasel)
