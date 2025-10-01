# Fiche Apprentissages

## Valider l’écriture du pet dans le fichier

Objectif:  
Assurer que la valeur générée par random_pet est bien écrite dans un fichier local, et pouvoir la vérifier via Terraform.
### Problème rencontré et pourquoi il est survenu

1. Post-condition insuffisante  
   - En essayant de vérifier directement l’écriture de la valeur via une postcondition dans la resource random_pet, il n’était pas possible de confirmer son écriture dans le fichier.  
   - Le bloc postcondition sur random_pet ne permet pas de lire le contenu écrit dans un fichier par une autre ressource local_file.  

2. Erreur de chemin dans le test  
   - Lors de l’utilisation du test pet_name_pattern, j’ai eu un souci avec la déclaration du source.  
   - J’avais mis ../ au lieu de ./, alors que le module de test se trouve au même niveau (Terraform prend aussi en compte le dossier test).  

3. Lifecycle mal utilisé  
   - J’ai tenté d’utiliser une postcondition dans le bloc lifecycle d’une datasource local_file.  
   - Le problème venait de la comparaison des attributs (self.content vs random_pet.pet.content) qui n’étaient pas correctement alignés.  

---

### Solution appliquée et pourquoi cette solution a fonctionné

1. Datasource local_file  
   - Après la création du fichier via resource "local_file", j’ai ajouté un data "local_file" pour rouvrir et relire le fichier.  
   - Cela permet à Terraform de comparer le contenu écrit avec la valeur générée par random_pet.  

   Exemple HCL:

   data "local_file" "pet_file" {
     filename = "./dist/pet.txt"
     lifecycle {
       postcondition {
         condition     = self.content == random_pet.pet.content
         error_message = "Le fichier ne contient pas le bon nom de pet généré."
       }
     }
   }

2. Test avec assert et regex  
   Dans le test pet_name_pattern, j’ai utilisé un assert avec une expression régulière pour valider le format du nom généré :

   assert {
     condition     = can(regex("^dev-[a-z]+$", output.pet_name))
     error_message = "Le nom (${output.pet_name}) ne respecte pas le pattern"
   }

   Correction apportée: mise du source = "./" au lieu de ../.

---

### Apprentissage et pourquoi c’est pertinent dans mon contexte

- Postcondition vs Datasource:  
  Une postcondition est utile pour vérifier la cohérence d’une ressource.  
  Une datasource permet de rouvrir et vérifier un état ou un fichier externe.  
  Dans ce cas, combiner les deux assure la vérification de bout en bout (génération + persistance).

- Précondition préférable:  
  Il est plus judicieux de valider les données d’entrée (préconditions et contraintes sur les variables) avant la création de la ressource.  
  Cela évite du temps perdu et limite les effets de bord sur d’autres ressources dépendantes.

- Précision des chemins dans Terraform:  
  L’organisation des modules et tests demande de bien comprendre les chemins relatifs ("./" vs "../").  
  Une petite erreur de chemin peut bloquer un test entier.

- Utilité des asserts en test:  
  Les tests Terraform (via assert) permettent de sécuriser le code en vérifiant les outputs générés.  
  Cela s’ajoute à la validation de la logique d’infrastructure.  