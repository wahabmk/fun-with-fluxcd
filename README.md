# fluxcd-practice

## To Deploy
To deploy staging, run the following if repo doesn't exist:
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

If repo already exists then run:
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

## Miscellaneous
1. There is nothing special needed to upgrade CRDs. To upgrade a CRD just bump its version and push the commit and flux will do the rest.