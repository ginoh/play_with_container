### ORAS

https://github.com/deislabs/oras


### CLI Install

#### macOS の場合
```
mkdir -p oras-install/
tar -zxf oras_0.7.0_*.tar.gz -C oras-install/
mv oras-install/oras /usr/local/bin/
rm -rf oras_0.7.0_*.tar.gz oras-install/
```

### pull

### push

#### manifests
```
curl -H "Accept: application/vnd.oci.image.manifest.v1+json" localhost:5000/v2/hello-artifact/manifests/v2
{"schemaVersion":2,"config":{"mediaType":"application/vnd.acme.rocket.config.v1+json","digest":"sha256:98df5c495df63132fb26e1066eb3bee4e4d72973da11b4e4e5
13ab11c15145b7","size":29},"layers":[{"mediaType":"application/vnd.acme.rocket.layer.v1+txt","digest":"sha256:d2a84f4b8b650937ec8f73cd8be2c74add5a911ba64
df27458ed8229da804a26","size":12,"annotations":{"org.opencontainers.image.title":"artifact.txt"}}]}
```