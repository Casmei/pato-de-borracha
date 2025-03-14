# Vulnerabilidades

## 1. Autenticação

O sistema utiliza o pacote Laravel Sanctum, uma biblioteca focada em autenticação para SPAs (Single Page Applications) e APIs.

❌ Atualmente, o sistema utiliza o método de autenticação Basic Auth, que é simples de implementar, mas possui várias limitações em termos de segurança e flexibilidade. O Basic Auth transmite as credenciais de forma não criptografada, o que torna o sistema vulnerável a ataques de "man-in-the-middle" (MiTM), especialmente se não for utilizado em conjunto com HTTPS. Além disso, esse método não possui suporte para controle de expiração de sessão ou revogação de credenciais, o que dificulta a implementação de práticas modernas de segurança. [Basic Auth](https://pt.stackoverflow.com/questions/254503/o-que-%C3%A9-basic-auth)



✅ Uma alternativa mais segura e robusta seria a utilização de Json Web Token (JWT).  O JWT é um padrão amplamente adotado para autenticação e troca de informações entre sistemas de forma compacta e segura. Ele permite armazenar não apenas informações de autenticação, mas também dados adicionais em formato JSON, e possui a vantagem  de ser assinado digitalmente. Isso significa que qualquer alteração no token pode ser  facilmente detectada. Além disso, o JWT permite definir um tempo de expiração, aumentando a segurança e permitindo a revogação de sessões com mais flexibilidade. Sua utilização é especialmente recomendada para ambientes distribuídos, como APIs RESTful, já que elimina  a necessidade de manter um estado de sessão no servidor, oferecendo escalabilidade e melhor controle sobre as sessões de usuários.

## 2. O sistema utiliza Raw SQL
> app/Repositories/User/Retrieve.php

❌ No repositório de usuários, existe uma classe responsável por listar os usuários com paginação. No entanto, essa implementação utiliza o método `whereRaw()`, permitindo a inserção de variáveis diretamente na query SQL. Embora isso possa ser útil em alguns casos, essa abordagem apresenta sérios riscos de segurança, pois ela abre portas para ataques de **SQL Injection**, uma das vulnerabilidades mais comuns em sistemas web. O uso de dados não validados diretamente nas consultas pode permitir que um atacante injete código SQL malicioso, comprometendo a integridade e segurança do banco de dados.

Exemplo do erro:
```php
$this->builder->whereRaw("name LIKE '%" . $this->name . "%'");
```

✅ Uma forma mais segura e recomendada de construir consultas no Laravel  é utilizando o método `where()` ou outros métodos de consulta do Eloquent,  que automaticamente escapam as variáveis, prevenindo SQL Injection. Um exemplo de implementação segura seria:
```php
$this->builder->where('name', 'LIKE', '%' . $this->name . '%');
```
Essa abordagem utiliza o "binding" de parâmetros, que é a maneira segura  de passar dados para as consultas SQL no Laravel, evitando que o valor seja interpretado como parte do código SQL, reduzindo significativamente o risco de injeções maliciosas.

## 3. Role Hard Code
> app/Policies/BasePolicy.php

❌ No código atual, as validações de cargo do usuário são feitas de forma  "hardcoded", com strings diretamente inseridas. Isso é problemático porque  pode gerar erros de manutenção, como a necessidade de atualizar o nome do  cargo em múltiplos lugares e o risco de falhas na política de permissões.

✅ A solução é usar **enums**, que centralizam os cargos e evitam erros de  digitação. Por exemplo, ao invés de usar `'MANAGER'` diretamente, podemos  definir um enum como:

```php
enum UserType: string {
    case MANAGER = 'MANAGER';
}
```

Isso torna o código mais seguro, legível e fácil de manter.

## 4. Dados sensíveis no error handling
> app/UseCases/User/Create.php

❌ No use case para criação de um usuário, é utilizado um bloco try-catch para capturar exceções durante o processo de criação. Caso ocorra um erro, a função `defaultErrorHandling()` é chamada, e os parâmetros do usuário (contidos em `CreateParams`), incluindo a senha, são passados para esse método. O problema é que, ao fazer isso, dados sensíveis, como a senha do usuário, podem ser registrados no log dentro do `defaultErrorHandling`. 

Esse tipo de prática representa um risco sério de **vazamento de dados sensíveis**. Registros de logs não devem armazenar informações críticas, como senhas, dados financeiros, ou qualquer outro dado pessoal e confidencial. Isso poderia ser explorado em caso de comprometimento do sistema de logs, expondo informações do usuário a potenciais atacantes.

Logar senhas, mesmo que em um ambiente de erro, pode colocar em risco a segurança da aplicação e violar as **melhores práticas de segurança** e regulamentos como a **LGPD (Lei Geral de Proteção de Dados)**

✅ Para evitar esse risco, é necessário **sanitizar os dados** antes de passá-los para os logs. Uma solução simples seria remover a senha dos parâmetros antes de passá-los para o `defaultErrorHandling`. Isso pode ser feito utilizando a função `unset()` para garantir que a senha não seja registrada. Além disso, é possível criar um **método de logging especializado** que permite fazer a sanitização dos dados automaticamente, removendo campos sensíveis como senhas antes de fazer qualquer log.

Alternativamente, se os dados do usuário não forem necessários para o log, a solução mais segura seria **não enviar os dados sensíveis para o log**. Em vez disso, seria mais adequado registrar apenas um identificador único ou uma referência ao erro, sem expor dados críticos.

## 5. Verificação de Permissões de Acesso

❌ Atualmente, o sistema realiza a verificação de forma inadequada, acessando as propriedades do usuário antes de verificar se ele está autenticado. Isso pode levar a erros de execução, especialmente quando o usuário não está logado, resultando em uma tentativa de acessar uma propriedade de um objeto `null`, o que provoca uma exceção.

✅ A correção mais adequada seria verificar primeiro se o usuário está autenticado antes de acessar suas propriedades.