# Injeção de Dependência no Controller

![alt text](image.png)
## 1. Problema na criação manual de dependências
> app/Http/Controllers/CompanyController.php
> Isso ocorre em vários outros lugares, esse ponto é aberta para uma discussão geral sobre o projeto

❌ No código atual do método `update`, as dependências são criadas manualmente dentro do método usando o operador `new`. Isso cria um acoplamento forte entre o controller e as implementações concretas (`UpdateDomain`, `CompanyUpdate` e `Company::find`). Essa abordagem torna o código difícil de testar, já que não é possível substituir essas dependências por mocks em testes unitários, além de violar o princípio de inversão de dependência do SOLID.

```php
$dominio = (new UpdateDomain(
    Auth::user()->company_id,
    $request->name,
))->handle();

(new CompanyUpdate($dominio))->handle();

$resposta = Company::find(Auth::user()->company_id)->first()->toArray();
```

✅ A solução recomendada é utilizar a injeção de dependência no construtor ou no método, permitindo que o framework resolva as dependências automaticamente. Isso melhora a testabilidade, seguindo os princípios SOLID, e facilita a substituição de implementações no futuro.

```php
public function __construct(
    private CompanyUpdateInterface $companyUpdate
) {}

public function update(UpdateRequest $request): JsonResponse
{
    // Nem tem sentido chamar o dominio aqui, já que ele não faz nada

    $companyData = $this->companyUpdate->execute($dominio);

    return $this->response(
        new DefaultResponse(
            new UpdateResource($companyData)
        )
    );
}
```

## 2. Falta de interfaces para abstrair implementações

❌ O código não utiliza interfaces para abstrair as implementações concretas. Isso limita a flexibilidade, dificulta testes unitários e cria um forte acoplamento com implementações específicas, violando o princípio de inversão de dependência.

✅ É recomendável criar interfaces para todos os serviços e repositórios. Por exemplo:

```php
interface CompanyUpdateInterface
{
    public function execute(UpdateDomain $domain): array;
}

class CompanyUpdate implements CompanyUpdateInterface
{
    public function __construct(
        private CompanyRepositoryInterface $repository
    ) {}

    public function execute(UpdateDomain $domain): array
    {
        // Implementação...
        return $updatedData;
    }
}
```

As interfaces permitem:
- Substituir implementações facilmente
- Aplicar mocks em testes
- Separar o contrato (o que o serviço faz) da implementação (como ele faz)

## 3. Container de Injeção de Dependência não utilizado
O Laravel possui um poderoso container IoC (Inversion of Control) que pode ser configurado no `AppServiceProvider` para associar interfaces a implementações concretas:

```php
// Em AppServiceProvider.php
public function register()
{
    $this->app->bind(CompanyUpdateInterface::class, CompanyUpdate::class);
    $this->app->bind(CompanyRepositoryInterface::class, EloquentCompanyRepository::class);
}
```

Isso permite que o código seja mais limpo, com menos acoplamento e mais fácil de testar e manter.

## 4. Gateway acoplado
❌ Eu senti que o acoplamento de serviços é um problema geral do código, em diferentes camadas, mas eu gostaria de comentar, em especifico, sobre a do Gateway, responsável pela conexão com serviços bancários externos. Este acoplamento dificulta a troca de Gateways (uma necessidade mais comum do que se imagina) de maneira rápida e segura. Em projetos anteriores que participei, estavamos enfrentando problemas com um gateway X, conseguimos migrar para outro simplesmente implementando uma nova classe concreta que respeitava o contrato definido pela interface, sem necessidade de alterações em outras partes do sistema.

✅ Implementar interfaces para abstrair os gateways permite:
- Substituir provedores de pagamento sem modificar o código de negócio
- Executar testes unitários com mocks de gateways
- Criar implementações específicas para diferentes cenários

```php
interface PaymentGatewayInterface
{
    public function processPayment(Order $order): PaymentResult;
    public function refund(string $transactionId): RefundResult;
    public function getInstallmentOptions(float $amount): array;
}

class ConcretePaymentGateway implements PaymentGatewayInterface
{
    // Implementação específica
}
```

### Smart Gateways (Uma ideia que eu tive)
Os Smart Gateways utilizam o padrão Factory para selecionar dinamicamente o gateway mais adequado com base em regras de negócio. Por exemplo: se um cliente precisa de parcelamento em 12x, mas o gateway principal suporta apenas 8x, o Smart Gateway pode redirecionar automaticamente para um gateway alternativo que ofereça essa condição.

```php
class PaymentGatewayFactory
{
    public function createGateway(Order $order): PaymentGatewayInterface
    {
        if ($order->installments > 8) {
            return new AlternativeGateway();
        }
        
        return new PrimaryGateway();
    }
}
```