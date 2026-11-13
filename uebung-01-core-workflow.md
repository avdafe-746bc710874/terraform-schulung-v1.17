# Ü1: Terraform Core Workflow kennenlernen

## Problemstellung

Wir sind frisch im Team und sollen ab nächster Woche Infrastruktur mit Terraform verwalten. Bevor wir echte Cloud-Ressourcen anlegen, wollen wir den Ablauf der Terraform-Befehle verstehen: **Was passiert wann, in welcher Reihenfolge, und woran erkennen wir, dass Terraform das Richtige tun wird?**

Wir arbeiten dafür bewusst ohne Cloud-Provider und nutzen die eingebaute Ressource `terraform_data`. Sie legt nichts in der Welt an, durchläuft aber denselben Lebenszyklus wie jede andere Ressource.

## Lernziele

Nach der Übung können wir:

- den vollständigen Terraform-Lebenszyklus (`init` → `plan` → `apply` → Änderung → `plan` → `apply` → `destroy`) eigenständig durchführen,
- die Ausgabe von `terraform plan` lesen und die Symbole (`+`, `~`, `-`) korrekt deuten,
- beschreiben, was `terraform init`, `plan`, `apply` und `destroy` jeweils tun, und was sie *nicht* tun.

## Arbeitsschritte

1. **Projekt anlegen.** Ein leeres Verzeichnis erstellen und eine Datei `main.tf` anlegen.
2. **Ressource schreiben.** Einen Block `resource "terraform_data" "..." { ... }` definieren. Als `input` einen beliebigen Wert setzen (z. B. einen String).
3. **`terraform init`** ausführen. Anschließend im Verzeichnis schauen, was neu entstanden ist.
4. **`terraform plan`** ausführen. Die Ausgabe gemeinsam lesen, bevor wir weitermachen. Was wird angelegt, was steht im Plan, was nicht?
5. **`terraform apply`** ausführen und bestätigen.
6. **`terraform plan` erneut** ausführen, *ohne* etwas am Code zu ändern. Was sagt Terraform jetzt?
7. **Wert im `input` ändern.**
8. **`terraform plan`**, wie unterscheidet sich die Ausgabe vom ersten Mal?
9. **`terraform apply`**, bestätigen und beobachten.
10. **`terraform destroy`** ausführen. Auch hier: Ausgabe vorher lesen.

Wenn wir am Ende Zeit haben: Einen zweiten `terraform_data`-Block hinzufügen und den Zyklus nochmal durchspielen.

## Weiterführende Links

- Terraform CLI Command Reference: <https://developer.hashicorp.com/terraform/cli/commands>
- Die Ressource `terraform_data`: <https://developer.hashicorp.com/terraform/language/resources/terraform-data>
- `terraform plan` im Detail: <https://developer.hashicorp.com/terraform/cli/commands/plan>
- Tutorial „Terraform Build Infrastructure": <https://developer.hashicorp.com/terraform/tutorials/cli>

## Reflexionsfragen

- Was passiert, wenn wir `terraform apply` ein zweites Mal ausführen, ohne den Code zu ändern? Warum ist das so und warum ist das eine nützliche Eigenschaft?
- Welche Dateien und Verzeichnisse hat `terraform init` angelegt? Wofür ist jedes davon da, und welche davon gehören ins Git-Repository, welche nicht?
- Wie lesen wir den `plan`-Output? Was bedeuten `+`, `~`, `-` jeweils, und an welcher Stelle erkennen wir, *wie viele* Ressourcen insgesamt betroffen sind?
- Warum sollten wir `terraform plan` anschauen, bevor wir `apply` ausführen, auch wenn `apply` den Plan ohnehin nochmal zeigt?
- `terraform_data` legt nichts in einer Cloud an. Welchen Zweck hat diese Ressource dann, und in welchen Situationen könnte sie in einem echten Projekt nützlich sein?
- Was würde ein versehentliches `terraform destroy` in einem Produktivprojekt anrichten? Welche Gewohnheit können wir uns jetzt schon angewöhnen, um das zu vermeiden?
- Wenn wir die Konfiguration löschen würden, ohne `destroy` aufzurufen, würde Terraform die Ressource wiederfinden? Warum (nicht)?
