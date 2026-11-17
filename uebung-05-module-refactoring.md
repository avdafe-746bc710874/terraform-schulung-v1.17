# Ü5: Refactoring in Module — `storage` und `access`

## Problemstellung

Am Ende von Ü4 haben wir festgestellt, dass der Code nicht mehr so übersichtlich ist wie in Ü2. Drei Buckets, Versioning, DynamoDB, IAM-Role, Policy, Attachment, Locals, Data Source, dev/prod-Unterscheidungen; inhaltlich alles richtig, aber **thematisch gemischt**: Datenhaltung und Zugriffsrechte stehen in derselben Datei.

Gleichzeitig zeichnet sich ab, dass ein zweites Projekt (anderer Kontext, gleiche Struktur) denselben Aufbau wieder brauchen wird. Copy-Paste wäre der falsche Weg.

Wir sollen den bestehenden Code daher in **zwei Module** aufteilen:

- ein `storage`-Modul — alles rund um die Daten (S3-Buckets, Versioning, DynamoDB),
- ein `access`-Modul — alles rund um die Rechte (IAM-Role, Policy, Attachment).

Eine optionale Randbedingung: **`terraform plan` muss vor und nach dem Refactoring identisch sein.** Wir verändern die Welt nicht, wir sortieren nur den Code um. Nichts wird zerstört, nichts neu angelegt.

## Lernziele

Nach der Übung können wir:

- bestehenden Code in **Module restrukturieren**
- die **Input-/Output-Schnittstelle** eines Moduls bewusst entwerfen und begründen, welche Werte nach außen gehören und welche nicht.

## Arbeitsschritte

**Wichtig:** Bevor wir anfangen, führen wir einmal `terraform plan` aus. Die Ausgabe sollte idealerweise „No changes" aufweisen.

1. **Modul-Verzeichnisse anlegen.** Unter `modules/storage/` und `modules/access/` jeweils `main.tf`, `variables.tf` und `outputs.tf`.
2. **Schnitt festlegen, bevor wir den Code verschieben.** Was gehört ins `storage`-Modul, was ins `access`-Modul? Wo verläuft die Grenze bei den Locals (`base_name`, `common_tags`)? Wer braucht die Data Source `aws_caller_identity`? Beide Module, nur eines, oder bleibt sie im Root?
3. **`storage`-Modul befüllen.** Die Bucket-Ressourcen (inkl. Versioning) und die DynamoDB-Tabelle ins Modul verschieben. Alles, was von außen kommen muss, wird als **Input-Variable** deklariert: mindestens `base_name`, `common_tags`, die Bucket-Map und das, was für dev/prod-Unterschiede nötig ist. Keine Werte hardcoden, die vorher aus Variablen oder Locals kamen.
4. **`storage`-Outputs definieren.** Überlegt: Was wird das `access`-Modul brauchen? Was könnte jemand außerhalb (z. B. eine später hinzukommende Lambda) brauchen? Gebt die Bucket-ARNs und -Namen als **Map** zurück, die DynamoDB-Werte einzeln.
5. **`access`-Modul befüllen.** IAM-Role, Policy und Attachment ins Modul verschieben. Die Bucket-ARNs, auf die die Policy sich bezieht, kommen als Input-Variable. Das Modul soll nur als Input die Bucket-Arns erhalten.
6. **`access`-Outputs definieren.** Mindestens die Role-ARN. Begründet: Warum ist gerade das der relevante Vertrag nach außen?
7. **Im Root-Modul die Module einbinden.** Zwei `module`-Blöcke, einer pro Modul. Die Outputs des `storage`-Moduls werden als Inputs ans `access`-Modul weitergereicht, dadurch entsteht wieder eine **implizite Abhängigkeit**, genauso wie zwischen Ressourcen in Ü2.
8. **`terraform init`** ausführen. Was ist diesmal anders als beim ersten `init` in Ü1/Ü2? Was hat Terraform jetzt registriert, das vorher nicht da war?
9. **`terraform plan`** Wir erwarten: Terraform möchte alle bisherigen Ressourcen zerstören und unter ihrer neuen Adresse neu anlegen. Erst lesen, **dann** weitermachen. Warum will Terraform das?
10. **`terraform apply`** ausführen.
11. **Root-Outputs durchreichen.** Die bisherigen Top-Level-Outputs (Bucket-Map, Role-ARN, DynamoDB-Werte) so anpassen, dass sie aus den Modul-Outputs kommen. Ergebnis von `terraform output` sollte identisch sein zu Ü4.

## Weiterführende Links

- Module — Übersicht und Aufbau: <https://developer.hashicorp.com/terraform/language/modules>
- Module-Inputs (`variables`) und -Outputs: <https://developer.hashicorp.com/terraform/language/modules/syntax>
- Module entwickeln — Standardstruktur: <https://developer.hashicorp.com/terraform/language/modules/develop/structure>
- `moved`-Block für Refactoring: <https://developer.hashicorp.com/terraform/language/modules/develop/refactoring>
- Module aufrufen (`module`-Block): <https://developer.hashicorp.com/terraform/language/modules/syntax#calling-a-child-module>
- Public Module Registry: <https://registry.terraform.io/browse/modules>
- Tutorial „Module Creation": <https://developer.hashicorp.com/terraform/tutorials/modules/module-create>

## Reflexionsfragen

- Welche **Inputs** hat am Ende unser `storage`-Modul, welche das `access`-Modul? Warum genau diese, und welche Werte haben wir bewusst **nicht** als Input durchgereicht, obwohl man es theoretisch könnte?
- Welche **Outputs** braucht das `access`-Modul vom `storage`-Modul? Wäre es sinnvoller gewesen, *alle* Modulwerte nach außen zu exportieren, nach dem Motto „schaden kann's nicht"? Warum (nicht)?
- Ein Kollege schlägt einen **anderen Modulschnitt** vor: ein Modul pro *Bucket-Typ* (archiv, reports, backups), jedes mit eigener Role. Was wäre daran problematisch? An welcher Stelle würde die Schnittstelle unangenehm?
- Stellt euch vor, ein zweites Projekt will denselben Aufbau: drei Buckets, eine Role, eine DynamoDB. Wie viel Code müsstet ihr jetzt **duplizieren**, wie viel könnt ihr wiederverwenden? Welches Modul ist projektspezifisch, welches generisch?
- Wir haben im Root-Modul die Outputs aus den Modulen durchgereicht. Warum nicht einfach `module.storage.bucket_arns` direkt dort nutzen, wo man die Werte braucht, und auf die Top-Level-Outputs verzichten? Für wen sind die Top-Level-Outputs eigentlich da?
- Die öffentliche Module Registry bietet fertige `terraform-aws-modules/s3-bucket/aws`-Module an. Unter welchen Kriterien würdet ihr so ein Fremdmodul verwenden, und unter welchen Kriterien lieber selbst eines schreiben, wie wir es gerade getan haben?
- Die optionale Randbedingung sagte uns der `terraform plan` nach dem Refactoring **musste** leer sein. Wie kann der `moved`-Blöcken dabei genutzt werden? Konkret: Was wäre mit unseren drei Buckets, ihren Daten, und den Beziehungen zur IAM-Role geschehen?
