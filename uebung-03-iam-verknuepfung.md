# Ü3: IAM-Verknüpfung für den Log-Archiv-Bucket

## Problemstellung

Unser Log-Archiv-Bucket aus Ü2 steht, aber im Moment könnte theoretisch jeder Service im AWS-Account darauf zugreifen, der passende Credentials besitzt. Das Security-Team besteht darauf, dass **nur berechtigte Services** auf den Bucket schreiben dürfen, und zwar über definierte IAM-Rollen, nicht über geteilte Zugangsschlüssel. Wir sollen eine IAM-Role anlegen, die ein Service (z. B. ein Log-Forwarder auf einer EC2-Instanz) annehmen kann, und ihr über eine Policy genau die Rechte geben, die sie für *unseren* Bucket braucht.

Nebenbei merken wir: Wir bauen gerade an mehreren Stellen ähnliche Namen zusammen. Das soll sauber und konsistent werden, bevor es mit jeder weiteren Ressource schlimmer wird.

## Lernziele

Nach der Übung können wir:

- Ressourcen über Attribut-Referenzen miteinander **verknüpfen** und den impliziten Abhängigkeitsgraph dadurch entstehen lassen,
- eine IAM-Role mit zugehöriger Policy anlegen und die Policy gezielt auf unseren Bucket **scharfstellen**,
- Outputs definieren, um Werte (ARNs, Namen) an spätere Module oder Konsument:innen **weiterzureichen**,
- ein konsistentes Namensschema **entwerfen** und quer über alle Ressourcen durchziehen,
- Terraform-Functions (insbesondere `jsonencode`) in der Doku **nachschlagen** und sinnvoll einsetzen.

## Arbeitsschritte

Wir erweitern das Projekt aus Ü2. Den Bucket legen wir **nicht** neu an. Unser Präfix ist: `8560d78348f5cc1e15-<namenskürzel>-` (z.B Max Mustermann -> `8560d78348f5cc1e15-mamu-`) bei allen Ressourcennamen verwenden.
1. **IAM-Role definieren.** Eine `aws_iam_role` anlegen. Die Assume-Role-Policy legt fest, *wer* die Rolle annehmen darf. Es reicht für uns zunächst ein Service-Principal (z. B. `ec2.amazonaws.com`). In der Doku schauen: Welche Felder sind Pflicht, welche optional?
2. **Policy-Dokument formulieren.** Eine `aws_iam_policy` anlegen, die Schreib-/Leserechte auf **genau unseren Bucket** erlaubt. Entscheidend: Der Bucket-ARN wird nicht als String hardcodiert, sondern als **Attribut-Referenz** auf die Bucket-Ressource geschrieben. Beachten: Objekte im Bucket brauchen einen eigenen ARN-Pfad (`<bucket-arn>/*`).
3. **Policy an Role binden.** Mit `aws_iam_role_policy_attachment` die Verbindung zwischen Role und Policy herstellen.
4. **Namensschema bleibt bestehen** Wir bleiben bei dem Schema (z. B. `<präfix>-<kürzel>-<umgebung>-<zweck>`) und wenden es konsistent auf Bucket-Name, Role-Name und Policy-Name an. Die Bausteine kommen aus unseren Variablen. Wo fällt uns auf, dass wir denselben Basisnamen mehrfach zusammenbauen?
5. **Outputs ergänzen.** Mindestens drei Outputs definieren: Bucket-Name, Bucket-ARN, Role-ARN. Überlegen, was wir nach außen sichtbar machen wollen.
6. **`terraform plan`** lesen. Der Output wird länger als in Ü2. Wir achten auf die **Reihenfolge**, in der Terraform die Ressourcen anlegen will.
7. **`terraform apply`** ausführen. Anschließend in der AWS-Konsole prüfen: Existiert die Rolle (IAM)? Ist die Policy daran angehängt? Enthält das Policy-Dokument tatsächlich den ARN unseres Buckets?
8. **Abhängigkeit erleben.** Den Bucket-Namen (`-archive`) ändern und `plan` ausführen (**nicht apply**). Was plant Terraform für Role und Policy? Warum? Wert danach wieder zurücksetzen.
9. **Outputs nutzen.** `terraform output` ausführen, die Role-ARN brauchen wir in den Reflexionsfragen.
10. **Nicht `destroy`en** — Ü4 baut auf diesem Stand auf.

## Weiterführende Links

- AWS Provider — `aws_iam_role`: <https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/iam_role>
- AWS Provider — `aws_iam_policy`: <https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/iam_policy>
- AWS Provider — `aws_iam_role_policy_attachment`: <https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/iam_role_policy_attachment>
- Output Values: <https://developer.hashicorp.com/terraform/language/values/outputs>
- References to Resource Attributes: <https://developer.hashicorp.com/terraform/language/expressions/references>
- Resource Dependencies (implizit vs. explizit): <https://developer.hashicorp.com/terraform/tutorials/configuration-language/dependencies>
- Terraform Functions — Übersicht: <https://developer.hashicorp.com/terraform/language/functions>
- `jsonencode`: <https://developer.hashicorp.com/terraform/language/functions/jsonencode>
- IAM Policy Elements (AWS): <https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_elements.html>

## Reflexionsfragen

- Wir haben nirgends `depends_on` geschrieben, und trotzdem legt Terraform die Ressourcen **in der richtigen Reihenfolge** an. Woran hat Terraform das erkannt? An welcher konkreten Stelle in unserem Code wird die Abhängigkeit zwischen Policy und Bucket sichtbar?
- Was wäre passiert, wenn wir den Bucket-ARN in der Policy **als String hardcodiert** hätten? In welchen Situationen fiele uns das später auf die Füße und woran hätten wir es gemerkt (oder gerade nicht)?
- Wenn wir die Role-Ressource aus dem Code entfernen und `apply` ausführen: Was plant Terraform für die Policy? Was für das `role_policy_attachment`? Warum genau in dieser Reihenfolge?
- Welche **drei Outputs** haben wir gewählt und warum gerade diese? Welcher davon wäre für ein späteres Modul (etwa eine Lambda, die Logs verarbeitet) am wichtigsten, welcher eher internes Debug-Material?
- Unser Namensschema zieht sich jetzt durch Bucket, Role und Policy. Was gewinnen wir damit für jemanden, der in sechs Monaten neu ins Projekt kommt? Und warum lässt sich **Ownership und Zweck** allein über Naming immer noch nicht vollständig abbilden. Wofür hatten wir in Ü2 die Tags?
- Für das Policy-Dokument haben wir (vermutlich) `jsonencode` verwendet. Was wäre eine Alternative gewesen?
- Angenommen, wir brauchen morgen einen **zweiten Bucket mit eigener Rolle** (z. B. für Backups): Welche Code-Teile müssten wir duplizieren? An welchen Stellen merken wir, dass unsere jetzige Struktur an ihre Grenze kommt?
- Wo haben wir in Schritt 4 gemerkt, dass wir denselben Basisnamen mehrfach zusammenbauen? Wäre es gut, wenn es dafür ein Konstrukt gäbe, mit dem wir solche abgeleiteten Werte **einmal** definieren und dann überall verwenden können? (Merken für Tag 2.)
- Die Assume-Role-Policy nennt aktuell einen konkreten Service-Principal. Was bedeutet das technisch: wer kann die Rolle annehmen, wer nicht? Und welches Risiko entstünde, wenn dort versehentlich `"*"` stünde?
