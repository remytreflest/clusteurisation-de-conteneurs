# Compte-rendu — TP Image Docker : Optimiser le poids

## Résultats

| Image | Taille |
|-------|--------|
| `tp-docker:classic` | 1.12 GB |
| `tp-docker:multistage` | 202 MB |
| **Gain** | **~82% de réduction** |

### Commandes pour observer la taille des images

```bash
# Lister toutes les images tp-docker avec leur taille
docker images
```

---

## Analyse des gains par étape de build

### Image `classic` — décomposition des layers (total : 1.12 GB)

| Étape | Layer | Taille |
|-------|-------|--------|
| Image de base Debian | `debian.sh bookworm` | 117 MB |
| Outils système (curl, gnupg...) | `apt-get update` + packages | 48 MB |
| Outils de build (gcc, make...) | `apt-get install build-essential` | 177 MB |
| Outils supplémentaires | `apt-get install` | **588 MB** |
| Installation de Node.js | `dpkg --install node` | 160 MB |
| Yarn | `set -ex && export GNUPGHOME` | 5 MB |
| `COPY package*.json` | — | 69 kB |
| `RUN npm ci` (deps prod + dev) | 158 packages installés | 16 MB |
| `COPY . .` (code + node_modules local) | — | 13 MB |
| `RUN npm run build` | placeholder | ~0 |

> Le vrai problème : les 813 MB de la base Debian complète, embarqués inutilement.

---

### Image `multistage` (stage runtime) — décomposition des layers (total : 202 MB)

| Étape | Layer | Taille | Gain vs classic |
|-------|-------|--------|-----------------|
| Image de base slim | `debian.sh bookworm slim` | **74.8 MB** | -42 MB |
| Installation de Node.js (slim) | `dpkg --install node` | **118 MB** | -42 MB |
| Yarn (slim) | `set -ex` | 7 MB | ~= |
| `COPY package*.json` | — | 69 kB | ~= |
| `RUN npm ci --omit=dev` + cache clean | 74 packages seulement | **2.47 MB** | **-13.8 MB** |
| `COPY server.js` | — | 314 B | — |
| `COPY public/` | — | 192 B | — |

> Les stages `deps` et `build` (node:20 complet) ne sont **pas inclus** dans l'image finale : ils sont utilisés puis jetés.

---

### Résumé visuel des gains

```
classic :   [== Debian complète 813MB ==][= Node 160MB =][npm 16MB][code 13MB]
                                                                        total: 1.12 GB

multistage: [= Debian slim 74MB =][= Node 118MB =][npm 2.5MB][code ~0]
                                                                        total: 202 MB

           ↑                       ↑                  ↑
        -739 MB               -42 MB              -13.8 MB
     (outils inutiles)   (Node allégé)    (devDeps exclues)
```

docker history tp-docker:classic
86e565721dc6   16 minutes ago   CMD ["npm" "run" "start"]                       0B        buildkit.dockerfile.v0
<missing>      16 minutes ago   EXPOSE map[3000/tcp:{}]                         0B        buildkit.dockerfile.v0
<missing>      16 minutes ago   RUN /bin/sh -c npm run build # buildkit         689B      buildkit.dockerfile.v0
<missing>      16 minutes ago   COPY . . # buildkit                             13.2MB    buildkit.dockerfile.v0
<missing>      16 minutes ago   RUN /bin/sh -c npm ci # buildkit                16.3MB    buildkit.dockerfile.v0
<missing>      16 minutes ago   COPY package*.json ./ # buildkit                69kB      buildkit.dockerfile.v0
<missing>      16 minutes ago   WORKDIR /app                                    0B        buildkit.dockerfile.v0
<missing>      4 days ago       CMD ["node"]                                    0B        buildkit.dockerfile.v0
<missing>      4 days ago       ENTRYPOINT ["docker-entrypoint.sh"]             0B        buildkit.dockerfile.v0
<missing>      4 days ago       COPY docker-entrypoint.sh /usr/local/bin/ # …   388B      buildkit.dockerfile.v0
<missing>      4 days ago       RUN /bin/sh -c set -ex   && export GNUPGHOME…   5.34MB    buildkit.dockerfile.v0
<missing>      4 days ago       ENV YARN_VERSION=1.22.22                        0B        buildkit.dockerfile.v0
<missing>      4 days ago       RUN /bin/sh -c ARCH= && dpkgArch="$(dpkg --p…   160MB     buildkit.dockerfile.v0
<missing>      4 days ago       ENV NODE_VERSION=20.20.2                        0B        buildkit.dockerfile.v0
<missing>      4 days ago       RUN /bin/sh -c groupadd --gid 1000 node   &&…   8.94kB    buildkit.dockerfile.v0
<missing>      13 days ago      RUN /bin/sh -c set -ex;  apt-get update;  ap…   588MB     buildkit.dockerfile.v0
<missing>      13 days ago      RUN /bin/sh -c set -eux;  apt-get update;  a…   177MB     buildkit.dockerfile.v0
<missing>      13 days ago      RUN /bin/sh -c set -eux;  apt-get update;  a…   48.4MB    buildkit.dockerfile.v0
<missing>      2 weeks ago      # debian.sh --arch 'amd64' out/ 'bookworm' '…   117MB     debuerreotype 0.17

docker history tp-docker:multistage

IMAGE          CREATED          CREATED BY                                      SIZE      COMMENT
b70a6b64f223   14 minutes ago   CMD ["node" "server.js"]                        0B        buildkit.dockerfile.v0
<missing>      14 minutes ago   EXPOSE map[3000/tcp:{}]                         0B        buildkit.dockerfile.v0
<missing>      14 minutes ago   COPY /app/public ./public # buildkit            192B      buildkit.dockerfile.v0
<missing>      14 minutes ago   COPY /app/server.js ./server.js # buildkit      314B      buildkit.dockerfile.v0
<missing>      14 minutes ago   RUN /bin/sh -c npm ci --omit=dev && npm cach…   2.47MB    buildkit.dockerfile.v0
<missing>      14 minutes ago   COPY package*.json ./ # buildkit                69kB      buildkit.dockerfile.v0
<missing>      14 minutes ago   ENV NODE_ENV=production                         0B        buildkit.dockerfile.v0
<missing>      14 minutes ago   WORKDIR /app                                    0B        buildkit.dockerfile.v0
<missing>      4 days ago       CMD ["node"]                                    0B        buildkit.dockerfile.v0
<missing>      4 days ago       ENTRYPOINT ["docker-entrypoint.sh"]             0B        buildkit.dockerfile.v0
<missing>      4 days ago       COPY docker-entrypoint.sh /usr/local/bin/ # …   388B      buildkit.dockerfile.v0
<missing>      4 days ago       RUN /bin/sh -c set -ex   && savedAptMark="$(…   7.18MB    buildkit.dockerfile.v0
<missing>      4 days ago       ENV YARN_VERSION=1.22.22                        0B        buildkit.dockerfile.v0
<missing>      4 days ago       RUN /bin/sh -c ARCH= OPENSSL_ARCH= && dpkgAr…   118MB     buildkit.dockerfile.v0
<missing>      4 days ago       ENV NODE_VERSION=20.20.2                        0B        buildkit.dockerfile.v0
<missing>      4 days ago       RUN /bin/sh -c groupadd --gid 1000 node   &&…   8.9kB     buildkit.dockerfile.v0
<missing>      2 weeks ago      # debian.sh --arch 'amd64' out/ 'bookworm' '…   74.8MB    debuerreotype 0.17
---

## Pourquoi le multi-stage réduit le poids

**1. Image de base plus légère (`node:20-slim` vs `node:20`)**
L'image `node:20` est basée sur Debian Bookworm complète (~1 GB : compilateurs, bibliothèques système, outils apt...). `node:20-slim` supprime la majorité de ces outils inutiles au runtime (~74 MB de base vs ~117 MB).

**2. Exclusion des devDependencies**
Avec `npm ci --omit=dev`, seules les dépendances de production sont installées dans l'image finale (74 packages au lieu de 158). Les outils de développement comme `eslint` et ses dépendances ne sont pas embarqués.

---

## Autres observations

- **`.dockerignore`** : exclure `node_modules` évite d'envoyer les modules locaux (compilés pour Windows) au daemon Docker lors du `COPY . .`, ce qui évite des bugs et accélère le transfert de contexte.
- **Cache Docker** : copier `package*.json` avant le reste du code permet de mettre en cache le layer `npm ci`. Si seul `server.js` change, les dépendances ne sont pas réinstallées.
- **Surface d'attaque réduite** : moins de binaires et de packages dans l'image finale = moins de CVE potentielles exposées en production.

---

## Amélioration à appliquer en production

Utiliser `node:20-alpine` au lieu de `node:20-slim` dans le stage runtime. Alpine Linux (~5 MB de base) permettrait de descendre à ~80-100 MB. Point de vigilance : Alpine utilise `musl libc` au lieu de `glibc`, ce qui peut créer des incompatibilités avec certains modules natifs.
