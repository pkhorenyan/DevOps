
```
terraform plan
terraform apply
terraform destroy -target <resource_type> <resource_name>
```

Как только переименовываем файл с переменными с terraform.tfvars во что-то другое, например terraform-dev.tfvars, команда `terraform apply` не поймет откдуа брать данные.

Если мы передаем переменные из конкретного файла:
```
terraform apply -var-file <file-name.tfvars>
```

Наши ключи хранятся в файле .aws/configure

Показать output:
```
terraform output
```

# Remote state

Показать атрибуты конкретного ресурса:
```
terraform state show <resource>
```

Показать все ресурсы в ремоут стейте:
```
terraform state list
```

Читает весь ремоут-стейт и выводит в stdout:
```
terraform state pull
```

Удаляет определенный ресурс из ремоут-стейта:
```
terraform state rm <resource>
```

Двигает/переименовывает определенный ресурс в ремоут-стейте.
Если вы изменили имя ресурса в коде (`resource "aws_instance" "new_name"`), Terraform решит, что старый нужно удалить, а новый создать.
```
terraform state mv <resource>
```

Вытаскивает из ремоут-стейта в локальный файл определенный ресурс:
```
terraform state mv -state-out="terraform.tfstate" <resource> <new_name_resource>
```

Перезаписывает ремоут-стейт:
```
terraform state push
```

