# Ubuntu 22.04 LTS <br> Installation de paquets système + pip sur machine offline






Ce guide décrit les méthodes d'installation de paquets sur machine x86_x64 Ubuntu 22.04 non connectée à internet [`cold`] :
 - les paquets système via apt
 - les paquets python via pip
 
Pour cela nous utilisons une seconde machine x86_x64, sous Ubuntu 22.04 également, connectée à internet  [`hot`] .
Vous pouvez utiliser une vm pour le rôle de  `hot`.
Ces méthodes fonctionnent probablement pour d'autres distributions basées sur Debian et d'autres architectures mais n'ont pas été testées.


| |   |
|:--------| :-------------:|
| Mise à jour| 07/07/2023 |
| Github| [ubuntu22.04-offline](https://github.com/runyournode/ubuntu22.04-offline)



## Prérequis : 
 - L'architecture de `hot` doit être la même que `cold`
 - être root sur les deux machines
 - disposer d'un moyen de transfert de données (clé USB, ssh, ...) entre les deux machines
 

# Installation de Paquets Système (apt)

## Sur hot 

 - S'assurer d'avoir configuré correctement la liste des repo si vous souhaitez installer des paquets non présents dans les repo officiels d'Ubuntu (par exemple `$ sudo add-apt-repository ppa:deadsnakes/ppa` pour les paquets python)
 - Télécharger le `<paquet>` ou les `<paquet1 paquet2 ...>` désirés ainsi que toutes leurs dépendances récursivement dans le repertoire courant :
	```bash
	hot: $ apt-get download $(apt-cache depends --recurse --no-recommends --no-suggests --no-conflicts --no-breaks --no-replaces --no-enhances <paquet1 paquet2 ...> | grep "^\w" | sort -u)
	```
 

## Sur cold

### A faire la première fois

 - Définissez un répertoire persistent qui sera votre repository local (exemple ici :  `~/local-apt-rep/`) 
	 ```bash
	cold: $ mkdir ~/local-apt-rep
	 ```

 - Editer le fichier `/etc/apt/sources.list` :
	```properties
	# Local repository
	deb [trusted=yes] file:<full_path_to_~/local-apt-repo> ./
	```
	Vous pouvez aussi commenter toutes les autres lignes pour éviter que apt perde tu temps à essayer de contacter les repo en ligne.

### A chaque installation de package

 - Copier les `*.deb` téléchargés depuis `hot` dans `~/local-apt-repo`.
 - Créer / Mettre à jour l'index du repo :
	```bash 
	cold: $ cd ~/local-apt-repo/
	cold: $ apt-ftparchive packages . > Packages
	```
- Mettre à jour apt :
	```bash 
	cold: sudo apt update
	```
 - Installer le ou les nouveaux paquets
	```bash
	cold: sudo apt install <paquet1 paquet2 ...>
	```

### Remarque : 
 Il est possible que apt vous renvoie le warning suivant, vous pouvez l'ignorer : 
`N: Download is performed unsandboxed as root as file '/host/download/apt/repo/./libpython3.11-minimal_3.11.4-1+jammy1_amd64.deb' couldn't be accessed by user '_apt'. - pkgAcquire::Run (13: Permission denied)`

# Installation de Paquets Python (pip)

## Sur hot
 - Créer un environnement virtuel basé sur la même version de python que la cible (python3.9 dans cet exemple):
	```bash
	hot: $ python3.9 -m venv ~/venv-3.9
	```
 - Activez l'environnement virtuel
	```bash
	hot: $ source ~/venv-3.9/bin/activate>	
	```
- Télécharger les paquets ainsi que leur dépendance dans le repertoire courant:
	```bash
	hot: $ pip download <paquet1 paquet2 ...>
	```
	
## Sur cold

### A faire la première fois
- Définissez un répertoire persistent qui sera votre repository local (exemple ici :  `~/local-pip-repo`) :
	 ```bash
	 cold: $ mkdir ~/local-pip-rep
	 ```
 - Créer/éditer le fichier de configuration de pip  `~/.config/pip/pip.conf` (vous pouvez vérifier si un fichier n'est pas déjà existant parmi  ` cold: $ pip config -v list`) :
	```properties
	[global]
	no-index = yes
	find-links = <full_path_to_~/local-pip-repo>
	```
### A chaque installation de package
	
 - Copier les `*.whl` téléchargés depuis `hot` dans `~/local-pip-repo`
 - Activer votre environnement virtuel
	```bash
	cold: $ source <path_to_venv/bin/activate>
	```
- Installer classiquement les paquets
	```bash
	cold: $ pip install <paquet1 paquet2 ...>
	```

### Remarque :
 - Pour gagner du temps, vous pouvez utiliser un fichier `requirement.txt` lors de ```$ pip download``` et ```$ pip install```
 - Si vous n'avez pas pris la peine de modifier le fichier de configuration, il faudra spécifier à chaque appel : 	
	```bash
	cold: $ pip install <paquet1 paquet2 ... --no-index --find-links ~/local-pip-repo
	```
- Vous pouvez éventuellement installer les paquets dans votre environnement virtuel sur `hot` afin d'avoir un clone de `cold` et limiter les conflits de paquets.   

# Mise en application : Installation d'un environnement de développement complet  PyTorch 2.0.1+cu118 (GPU enable)
 Installation de :
  - python3.9 (à ce jour - *i.e.* 06/2023 - pytorch supporte jusqu'à python3.10)
  - cuda 12.2 et son driver nvidia
  - pytorch 2.01+cu118

## Installation de python3.9 sur `hot`et `cold`
 ```bash
 hot: $ sudo apt update
 hot: $ sudo add-apt-repository ppa:deadsnakes/ppa
 hot: $ sudo apt install python3.9-full -y
 hot: $ mkdir ~/local-apt-repo && cd ~/local-apt-repo
 hot: $ apt-get download $(apt-cache depends --recurse --no-recommends --no-suggests --no-conflicts --no-breaks --no-replaces --no-enhances python3.9-full | grep "^\w" | sort -u)
```

Apres avoir configuré votre repository local pour apt sur `cold` et transféré les `.deb`: 
```bash
cold: $ cd ~/local-apt-repo/
cold: $ apt-ftparchive packages . > Packages
cold: $ sudo apt update && sudo apt install python3.9-full -y
```
## Installation de cuda 12.2 sur `cold`
pytorch2.0.1+cu118 nécéssite au moins la version de cuda 11.8, mais fonctionne avec les versions ultérieures. Nous installons donc la dernière version de cuda disponible à ce jour : cuda 12.2.
Vous pouvez également vous aider des deux sources officielles de nvidia suivantes :
 - [Installation Guide](https://docs.nvidia.com/cuda/cuda-installation-guide-linux/index.html)
 - [Release Note](https://docs.nvidia.com/cuda/cuda-toolkit-release-notes/index.html)

### Prérequis
- [x] Le gpu est bien reconnu : `cold: $ lspci | grep -i nvidia` doit retourner votre gpu
- [x] gcc est installé : `cold: $ gcc --version` doit retourner `11.3.0` minimum
- [x] Votre noyau est compatible : `cold: $ uname -r` doit retourner `5.19.0-38`
> Cette version ne semble plus disponible pour Unbuntu et nvidia ne précise pas l'ensemble des noyaux compatible, testé avec `linux-image-5.19.0-46-generic`
- [x] Le paquet linux-headers correspondant à votre noyau est installé : `cold: $ dpkg -l | grep linux-headers`
> Vous pouvez donc installer les paquets apt `linux-image-5.19.0-46-generic`,  `linux-headers-5.19.0-46-generic` et `gcc-11`.


### Installation des drivers nvidia + cuda
Chaque version de CUDA nécessite une version minimale de driver nvidia mais fonctionne avec les versions ultérieures.
 - Le plus simple est de telecharger le ```runfile (local)``` de CUDA Toolkit 12.2 car il contient cuda ainsi que son driver nvidia compatible.
	 - Pour notre cible : 
	`https://developer.download.nvidia.com/compute/cuda/12.2.0/local_installers/cuda_12.2.0_535.54.03_linux.run`
	  - Alternativement, vous pouvez télécharger une version spécifique sur le site de téléchargement de nvidia: [cuda last](https://developer.nvidia.com/cuda-downloads) , [cuda 11.8](https://developer.nvidia.com/cuda-11-8-0-download-archive).
- Copier `cuda_12.2.0_535.54.03_linux.run` à la racine du système `cold`.
- Fermer tous les programmes/fenêtres puis désactiver temporairement l'environnement graphique en passant au runlevel 3:
	```bash
	cold: $ sudo init 3
	```
- Se logger à nouveau (si besoin vous pouvez ouvrir un nouveau tty à l'aide de `ctrl`+`alt`+`F<n>`).
 - Installer cuda et son driver :
	```bash
	cold: $ sudo sh /cuda_12.2.0_535.54.03_linux.run
	```
	>  Il est recommandé de désinstaller avant les éventuels drivers nvidia ou la précédente installation cuda.
	>  Après redémarrage, vous pouvez vérifier l'installation avec la commande ```cold: $ nvidia-smi``` qui devra indiquer le driver nvidia ainsi que la version de cuda installée.
	

### Remarque
 - :warning: *non testé*  :  Vous pouvez aussi [installer cuda à travers pip](https://docs.nvidia.com/cuda/cuda-installation-guide-linux/index.html#pip-wheels), mais il faudra alors s'assurer  que le [driver nvidia compatible](https://docs.nvidia.com/cuda/cuda-toolkit-release-notes/index.html#id4) est déjà installé (`>=525.60.13`pour notre cas), par exemple [535.54.03](https://fr.download.nvidia.com/XFree86/Linux-x86_64/535.54.03/NVIDIA-Linux-x86_64-535.54.03.run)
 - :warning: **(A Confirmer!)**  Apres avoir mis à jour le driver nvidia, il est possible que votre machine refuse de `Suspend` ou `Power Off`. Ceci est dû à un bug de liens symboliques. Vous pouvez tenter l'action corrective :
	 ```bash
	cold: $ sudo rm /etc/systemd/system/systemd-hibernate.service.requires/nvidia-resume.service
	cold: $ sudo rm /etc/systemd/system/systemd-hibernate.service.requires/nvidia-hibernate.service
	cold: $ sudo rm /etc/systemd/system/systemd-suspend.service.requires/nvidia-resume.service
	cold: $ sudo rm /etc/systemd/system/systemd-suspend.service.requires/nvidia-suspend.service
	```

## Installation de torch2.0.1+cu118 sur `cold`
 - Activer votre environnement virtuel sur `hot` et télécharger les paquets dans le repertoire courant:
	```bash
	hot: $ source ~/venv-3.9/bin/activate
	hot: $ pip download torch torchvision torchaudio --index-url https://download.pytorch.org/whl/cu118
	```
	> Vous pouvez trouver la dernière version sur le site de [PyTorch](https://pytorch.org/get-started/locally/)
	> Cette commande ne fonctionnant pas sur ma vm `hot`, je l'ai effectuée depuis une machine physique. 
 - Apres avoir configuré votre repository local pour pip sur `cold` et transféré les `.whl`, installer les paquets sur `cold` dans votre environnement virtuel cible:
	```bash
	cold: $ source ~/venv-3.9-torch2.0.1/bin/activate
	cold: $ pip install torch torchvision torchaudio    
	```
	> Vous pouvez ensuite [vérifier l'installation](https://pytorch.org/get-started/locally/#windows-verification)
	 
