# TP Part 02 - CI/CD

## 2-1 : What are testcontainers ?

C'est une librairie qui permet de créer automatiquement des containers docker pour tester.

## 2-2 : Document your Github Actions configurations.

J'ai commencé par ajouter mes credentials pour Docker hub dans Github Action.
On les mets dans l'onglet secret afin de ne pas les écrire en dur dans le code.

La ligne 'needs: build-and-test-backend' permet de s'assurer que le backend fonctionne avant de pull et build les images, ce qui peut prendre assez longtemps.
Sans cette ligne s'il y a une erreur dans le backend on ne le saura pas avant la fin.