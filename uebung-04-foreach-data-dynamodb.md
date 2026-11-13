# Ü4: for_each, Data Sources und DynamoDB

## Problemstellung

Unser Log-Archiv-Bucket aus Ü2/Ü3 hat sich bewährt und jetzt wachsen die Anforderungen:
Zwei weitere Buckets kommen dazu, einer für generierte **Reports**, einer für nächtliche **Backups**. Strukturell sind sich alle drei ähnlich (Tags, Naming, Encryption), unterscheiden sich aber in Details (Zweck, Retention). Zusätzlich braucht das Team eine **DynamoDB-Tabelle** für Metadaten zu den abgelegten Objekten: wer hat wann was hochgeladen.

Nebenbei fällt uns zweierlei auf:

- Unser Code baut denselben **Basisnamen** inzwischen an drei, vier Stellen manuell zusammen (Bucket, Role, Policy). Das hatten wir am Ende von Ü3 schon gemerkt. Wie helfen uns `locals` ?
- Für IAM-Bedingungen und Tags brauchen wir die **AWS-Account-ID**. Hardcoden wollen wir sie nicht, als Variable wäre sie redundant (sie ergibt sich aus den Credentials).

Und schließlich: In **dev** und **prod** soll sich der Aufbau leicht unterscheiden, z. B. Bucket Versioning und Retention einzelner Objekte. Das wollen wir deklarativ lösen, nicht über zwei getrennte Code-Bäume.

## Lernziele

Nach der Übung können wir:

- eine Menge strukturähnlicher Ressourcen per `for_each` erzeugen und **begründen**, warum `for_each` hier die passendere Wahl ist als `count`,
- **Locals** von **Variables** abgrenzen und abgeleitete Werte (Namen, Tags) zentral an einer Stelle berechnen,
- eine **Data Source** nutzen, um Werte aus AWS zu lesen, ohne sie zu verwalten,
- einen Refactoring-Schritt ausführen, bei dem sich die Terraform-Adresse einer Ressource ändert, **ohne** die Ressource zu zerstören (`moved`-Block),
- Umgebungsunterschiede (dev/prod) über Locals oder Maps abbilden, statt den Code zu duplizieren,
- eine `aws_dynamodb_table` eigenständig aus der Provider-Doku konfigurieren.

## Arbeitsschritte

Wir erweitern das Projekt aus Ü3.
0. Unser Präfix: `8560d78348f5cc1e15-<namenskürzel>-` (z.B Max Mustermann -> `8560d78348f5cc1e15-mamu-`) bei allen Ressourcen verwenden.
1. **Locals einführen.** Das, was wir in Ü3 mehrfach zusammengebaut haben, ziehen wir jetzt in Locals hoch: einen `base_name` (aus Präfix, und Kürzel) und `common_tags` (owner, environment, ManagedBy ...). Ab jetzt greifen wir an den bestehenden Ressourcen auf `local.base_name` und `local.common_tags` zu und nicht mehr auf zusammengestückelte Strings.
2. **Account-ID per Data Source holen.** `aws_caller_identity` als Data Source einbinden und die Account-ID in `common_tags` aufnehmen (z. B. als `AccountId`). In der Provider-Doku nachschauen, welche Attribute die Data Source zurückgibt.
3. **Bucket-Varianten deklarieren.** Eine Map definieren, die die drei Buckets beschreibt: Keys: `archiv`, `reports`, `backups`. Pro Eintrag mindestens ein Attribut `retention_days`, welche wir in Schritt 5 benötigen. Diese Map lebt als `local` oder als Variable.
4. **Bucket-Resource auf `for_each` umstellen.** Die bisher einzelne `aws_s3_bucket`-Ressource auf `for_each = local.buckets` (oder `var.buckets`) umbauen. Bucket-Namen und Tags aus `base_name`, dem Map-Key und dem Map-Wert zusammensetzen.
5.  **Objekte im Bucket nach Tag X löschen** folgenden Ressource Block auf unser Projekt anpassen, sodass je nach Bucket die Objekte gelöscht werden; `reports` nach 30 Tagen, `archiv` nach 180 Tagen, `backups` nach 120 Tagen.
```
resource "aws_s3_bucket_lifecycle_configuration" "expire_object_after" {
  bucket = aws_s3_bucket.bucket.bucket

  rule {
    id = "expire-after"
    status = "Enabled"

    expiration {
      days = 30
    }
  }
}
```
6. **`terraform plan` lesen** Was will Terraform jetzt mit unserem bestehenden Bucket machen? Warum?
7. **`moved`-Block einfügen.** Terraform kennt Ressourcen nur über ihre Adresse, diese hat sich durch den for_each geändert. Mit einem `moved`-Block teilen wir Terraform mit, dass die alte Adresse in die neue überführt wird. Danach erneut `plan`: Der bestehende Bucket darf nur als *moved* auftauchen, nicht als *destroy/create*.
8. **IAM auf die neue Struktur anheben.** Die Policy aus Ü3 erlaubt Zugriffe derzeit nur auf *einen* Bucket. Passt sie so an, dass sie die ARNs *aller* Buckets enthält, nutzt hierfür eine For-Expression. Überlegt kurz: Wollen wir **eine** Role für alle Buckets oder **pro Bucket eine Role**?
9. **DynamoDB-Tabelle anlegen.** Eine `aws_dynamodb_table` für die Metadaten. Provider-Doku lesen: Welche Felder sind Pflicht (Name, Hash-Key, Billing-Mode, Attributes)? Tabelle mit `common_tags` versehen.
   ```
   resource "aws_dynamodb_table" "_" {
    ...

    billing_mode = "PAY_PER_REQUEST"
    hash_key = "object_data"
    attribute {
        name = "object_data"
        type = "S"
    }
    ...
   }
   ```
10. **dev/prod-Unterschiede einbauen.**
   - Bucket-**Versioning** (prod: an, dev: aus),
   - Bucket-Objekt Retention (aus Schritt 5),
   - DynamoDB nur auf prod erstellen lassen
   Entweder über `local`s mit `var.environment == "prod"` abgeleitet oder über eine Map pro Environment. Welche Variante ist lesbarer?
11. **`plan` lesen, `apply` ausführen.** Reihenfolge der Aktionen im Plan begründen. In der AWS-Konsole kurz prüfen: Drei Buckets da? Policy enthält alle drei ARNs? DynamoDB-Tabelle existiert, (nicht)?
12. **Outputs aktualisieren.** Die Bucket-Outputs als **Map** zurückgeben (Schlüssel → ARN / Name), nicht mehr als Einzelwert. DynamoDB-Tabellenname und -ARN als Outputs ergänzen.
13. **Nicht `destroy`en** Ü5 refactort auf diesem Stand weiter.

## Weiterführende Links

- `for_each` für Ressourcen: <https://developer.hashicorp.com/terraform/language/meta-arguments/for_each>
- `if` Filtering Elements: <https://developer.hashicorp.com/terraform/language/expressions/for#filtering-elements>
- Vergleich `count` vs. `for_each`: <https://developer.hashicorp.com/terraform/language/meta-arguments/count>
- Local Values: <https://developer.hashicorp.com/terraform/language/values/locals>
- Data Sources — Überblick: <https://developer.hashicorp.com/terraform/language/data-sources>
- `aws_caller_identity`: <https://registry.terraform.io/providers/hashicorp/aws/latest/docs/data-sources/caller_identity>
- `moved`-Block (Refactoring ohne Re-Create): <https://developer.hashicorp.com/terraform/language/modules/develop/refactoring>
- For-Expressions (Listen, Maps transformieren): <https://developer.hashicorp.com/terraform/language/expressions/for>
- AWS Provider — `aws_dynamodb_table`: <https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/dynamodb_table>
- AWS Provider — `aws_s3_bucket_versioning`: <https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/s3_bucket_versioning>

## Reflexionsfragen

- Warum haben wir `for_each` und nicht `count` gewählt? Was wäre passiert, wenn wir `count` genommen und dann einen Bucket aus der Mitte entfernt hätten? An welcher konkreten Code-Stelle wird der Unterschied zwischen den beiden Ansätzen sichtbar?
- Wir haben in Schritt 1 **Locals** eingeführt und in Schritt 3 eine **Map** für die Buckets angelegt, diese konnte `local` oder `var` sein. Nach welchem Kriterium habt ihr entschieden?
- In Schritt 6 wollte Terraform unseren bestehenden Bucket destroyen und neu anlegen. Warum, obwohl sich inhaltlich nichts geändert hat? Was sagt das über die **Identität** einer Ressource aus Terraform-Sicht?
- Wofür haben wir die Data Source `aws_caller_identity` benutzt? Warum ist sie hier **besser** als eine Input-Variable, in die wir die Account-ID selbst eintragen? Nennt ein weiteres Szenario in eurer realen Infrastruktur, in dem eine Data Source das passendere Werkzeug wäre als eine Ressource.
- Ihr habt in Schritt 8 zwischen *einer Rolle für alle Buckets* und *einer Rolle pro Bucket* entschieden. Was spricht für welche Variante? In welcher Situation würdet ihr die andere wählen? (Stichwort: Least Privilege vs. Komplexität.)
- Die dev/prod-Unterschiede haben wir in Schritt 10 entweder per `if`-Ausdruck in Locals oder per Map-Lookup abgebildet. Vergleicht die beiden Ansätze: Welcher skaliert besser, wenn morgen **staging** und **preprod** als weitere Umgebungen dazukommen?
- Wenn jemand die DynamoDB-Tabelle direkt in der AWS-Konsole löscht, was zeigt das nächste `terraform plan`? Und wie verhält sich das zu dem, was wir in Ü2 beim Thema Drift gesehen haben?
