# Ü2: S3-Bucket mit Variablen

## Problemstellung

Unsere Firma hat Log-Archivierung bisher informell gelöst. Jeder speichert Server-Logs dort, wo gerade Platz ist. Das soll sich ändern: Wir sollen einen **zentralen S3-Bucket für Log-Archive** in AWS bereitstellen, sauber mit Tags versehen, damit Kostenabrechnung und Ownership nachvollziehbar sind. Absehbar wird es nicht bei einem Bucket bleiben, also schreiben wir den Code von Anfang an so, dass wichtige Werte nicht hardcodiert sind.

## Lernziele

Nach der Übung können wir:

- eine reale AWS-Ressource (`aws_s3_bucket`) über Terraform anlegen und iterativ ändern,
- Input-Variablen mit Typ und Default-Wert definieren und an Ressourcen-Argumente anbinden,
- Tags als Mittel zur Dokumentation von Ownership und Zweck einsetzen,
- die `plan`-Ausgabe einer Änderung (`~`) von der eines Neuanlegens (`+`) unterscheiden,
- gezielt in der Provider-Doku recherchieren, welche Argumente eine Ressource kennt.

## Arbeitsschritte
Unser Präfix ist: `8560d78348f5cc1e15-<namenskürzel>-` (z.B Max Mustermann -> `8560d78348f5cc1e15-mamu-`) bei allen Ressourcennamen verwenden.

1. **Projekt anlegen.** Frisches Verzeichnis, darin die Dateien, die wir brauchen werden (z. B. `main.tf`, `variables.tf`, `default.auto.tfvars`, `versions.tf`).

```versions.tf
terraform {
  required_version = "~>1.17.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 6.0"
    }
  }
}

provider "aws" {
  region     = "eu-central-1"
  access_key = "<aus-der-mail-zu-entnehmen>"
  secret_key = "<aus-der-mail-zu-entnehmen>"
}
```
2. **Bucket definieren.** Einen Block `resource "aws_s3_bucket" "..." { ... }` schreiben. In der Provider-Doku nachschlagen: Welche Argumente sind Pflicht, welche optional? **Wichtig:** S3-Bucket-Namen sind global eindeutig, verwendet den Präfix.
3. **Variablen einführen.** Mindestens den Bucket-Namen und die Umgebung (`dev`/`test`/...) als Variablen herausziehen. Typen und sinnvolle Defaults setzen. Konkrete Werte landen in der `default.auto.tfvars`. Resultierender Name: `8560d78348f5cc1e15-<namenskürzel>-<umgebung>-<zweck>`
4. **Tags ergänzen.** Mindestens `owner`, `environment`, `application` als Tags an den Bucket hängen. `environment` sollte aus einer Variable kommen.
5. **`terraform init`** ausführen. Was ist diesmal im `.terraform`-Verzeichnis passiert, das in Ü1 noch nicht da war?
6. **`terraform plan`** lesen, dann **`terraform apply`** ausführen. Den Bucket anschließend kurz in der AWS-Konsole anschauen.
7. **Iterieren.** Einen Tag-Wert ändern (z. B. `application`). `plan` ausführen und genau schauen, wie sich die Ausgabe vom ersten Mal unterscheidet. Danach `apply`.
8. **State anschauen.** Mit `terraform show` in den State blicken. Was steht dort, das wir gar nicht explizit konfiguriert haben? Woher kommen diese Werte?
9. **Nicht `destroy`en** wir bauen in Ü4 auf diesem Bucket auf.

## Weiterführende Links

- AWS Provider — `aws_s3_bucket`: <https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/s3_bucket>
- Input Variables: <https://developer.hashicorp.com/terraform/language/values/variables>
- Variable Definition Files (`.tfvars`): <https://developer.hashicorp.com/terraform/language/values/variables#variable-definitions-tfvars-files>
- `terraform show`: <https://developer.hashicorp.com/terraform/cli/commands/show>
- AWS Tagging-Strategien: <https://docs.aws.amazon.com/general/latest/gr/aws_tagging_strategies.html>

## Reflexionsfragen

- Welche Werte in unserem Code sind **noch hardcodiert**, obwohl sie sich im nächsten Projekt vermutlich ändern? Wo ziehen wir die Grenze und was lohnt sich als Variable, was wäre übertrieben?
- Vergleich `plan` bei der ersten vs. bei der zweiten Änderung: Woran erkennen wir, dass die zweite Änderung **kein Neuanlegen** ist? Warum wäre dieser Unterschied in einem Produktivsystem kritisch?
- Im State stehen Felder, die wir nie konfiguriert haben (ARN, Region, Hosted-Zone-ID …). Woher stammen diese Werte, und wofür könnten sie in den nächsten Aufgaben nützlich werden?
- Wenn wir morgen einen **zweiten Bucket** brauchen (z. B. für Backups): Was müssten wir an unserem jetzigen Code konkret ändern? An welchen Stellen würde das unangenehm oder fehleranfällig?
- Die `default.auto.tfvars` enthält unsere konkreten Werte. Was gehört in diese Datei, was nicht? Welche Dateien gehören ins Git-Repository, und was passiert, wenn wir versehentlich Secrets dort ablegen?
- Wir haben Tags gesetzt. Was gewinnen wir damit jetzt und in sechs Monaten, wenn fünf Teams im gleichen AWS-Account arbeiten? Warum lässt sich das über reines Naming **nicht** vollständig lösen?
- Angenommen, jemand ändert einen Tag direkt in der AWS-Konsole. Was würde ein anschließendes `terraform plan` zeigen, und warum ist genau dieses Verhalten der Grund, warum wir Terraform überhaupt einsetzen?
