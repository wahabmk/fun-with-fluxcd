# fluxcd-practice

If repo doesn't exist:
```sh
flux bootstrap github \
--token-auth \
--owner=wahabmk \
--repository=git@github.com:wahabmk/fun-with-fluxcd.git \
--branch=main \
--path=clusters/staging \
--personal \
--verbose
```

If repo already exists then:
```sh
flux bootstrap git \
--token-auth=true \
--url=https://github.com/wahabmk/fun-with-fluxcd \
--username=wahabmk \
--password=<PAT> \
--branch=main \
--path=clusters/staging \
--verbose
```