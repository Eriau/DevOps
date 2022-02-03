# DevOps

## TP1 : Docker

https://github.com/Eriau/DevOps/blob/main/TP1/README.md

## TP2 : CI/CD

https://github.com/Eriau/DevOps/blob/main/TP2/README.md

**!!!** : Au niveau des tags sur les images, il faudrait aller les chercher soit dans le pom.xml pour l'api Java, soit sur les tags de Git, si y a un nouveau tag on le change, sinon on publie en latest.

## TP3 : Ansible

https://github.com/Eriau/DevOps/blob/main/TP3/README.md

**!!!** : Lorsque l'on fetch les images, si la version n'a pas changée il n y aura aucun pull. Donc soit supprimer les images et les containers à chaque fetch, soit faire un versionning bon.
**!!!** : Plutôt utiliser 'workflow_call' au lieu de 'workflow_run' c'est plus logique.

## TP_Extras

https://github.com/Eriau/DevOps/blob/main/TP_Extras/README.md