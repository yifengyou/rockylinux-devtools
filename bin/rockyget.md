

```
#!/bin/sh

NAME="${1:-$PACKAGE}"
BRANCH="${2:-${BRANCH:-r8}}"

set -e

usage() {
    echo "usage: rockyget PACKAGE [BRANCH]"
    echo "BRANCH (r8, r9) implies VERSION (8,9)"
}

if [ -z "${NAME}" ]; then
    usage
    exit 1
fi

if [[ "${NAME}" =~ ^rocky- ]]; then
    AARGS='--import-branch-prefix=r --rpm-prefix=https://git.rockylinux.org/original/rpms'
fi


case $BRANCH in
  r8*) 
    CDN_URL="https://rocky-linux-sources-staging.a1.rockylinux.org" ;;
  r9*)
    CDN_URL="https://sources.build.resf.org" ;;
  *)
    usage; exit 1;;
esac

PATCHPATH=${HOME}/rocky/patch/${NAME}.git
git config --global user.name "${GIT_USER:-devtool importer}"
git config --global user.email "${GIT_EMAIL:-releng+importer@rockylinux.org}" 
if [ ! -d "${PATCHPATH}" ]; then
    echo "Trying to clone existing patch repo"
    if curl -sI https://git.rockylinux.org/staging/patch/${NAME} | grep "HTTP/1.1 200 OK" >/dev/null 2>&1; then # if it's a real repo
        echo "Found existing patch repo. Cloning to ${PATCHPATH}"
    	git clone --branch ${BRANCH} https://git.rockylinux.org/staging/patch/${NAME}.git "${PATCHPATH}" ||:
    else
        echo "No patch repo exists. Initing empty"
        git init ${PATCHPATH}
        git -C ${PATCHPATH} checkout -b ${BRANCH} 
        git -C ${PATCHPATH} commit --allow-empty -m 'Initial commit' 
        git -C ${PATCHPATH} remote add origin https://git.rockylinux.org/staging/patch/${NAME}.git
    fi
fi

test -d "$HOME/rocky/rpms/${NAME}" && rm -rf "$HOME/rocky/rpms/${NAME}"
test -d /tmp/srpmproc-cache || mkdir /tmp/srpmproc-cache
mkdir -p "$HOME/rocky/rpms/${NAME}"

if [[ ! -f $HOME/.ssh/id_rsa ]]; then
  ssh-keygen -t rsa -q -f "$HOME/.ssh/id_rsa" -N ""
fi

pushd $HOME/rocky/rpms/ >/dev/null
srpmproc --cdn-url $CDN_URL  --version ${BRANCH:1:1} --upstream-prefix "file://$HOME/rocky" --storage-addr file:///tmp/srpmproc-cache --source-rpm "$NAME" --tmpfs-mode "${NAME}" ${AARGS}
popd >/dev/null

echo "Package has been pulled to: $HOME/rocky/rpms/${NAME}"
echo
echo "The following branches are available:"
( cd $HOME/rocky/rpms/${NAME}; ls -1 )
```


