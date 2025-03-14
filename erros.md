# Erros
## 1. Nome dos resources de forma geral
❌ O nome da classe `ShowResource` é genérico e não indica claramente a entidade que está sendo manipulada. Isso pode gerar confusão e dificultar a manutenção.
 
✅ Utilizar nomes mais específicos, como `ShowCompany`, torna a classe mais autoexplicativa e facilita a compreensão do seu propósito sem depender da estrutura de pastas.

## 2. Métodos que não são utilizados
> app/Repositories/BaseRepository.php

❌ Há métodos no código que não são utilizados, o que pode gerar confusão e aumentar a complexidade do código desnecessariamente.
 
✅ Remover métodos não utilizados para manter o código mais limpo, legível e fácil de manter.

## 3. Documentação errada
> app/Http/Controllers/CardController.php

❌ Na documentação, é informado que o método register é para ativar, o que está incorreto, e também no show a dos colocou POST, porém nas rotas está como verbo http GET
 
✅ Colocar na documentação as reais funcionalidades dos métodos