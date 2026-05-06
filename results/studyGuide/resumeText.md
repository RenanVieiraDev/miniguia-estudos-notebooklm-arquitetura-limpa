Este guia foi elaborado para levar você do zero ao entendimento prático da Arquitetura Limpa, utilizando os princípios de Robert C. Martin (Uncle Bob). O domínio utilizado será sempre uma API de Cadastro de Usuários.

--------------------------------------------------------------------------------
1. Visão Geral da Arquitetura Limpa
O que é: É um padrão que organiza o código em camadas para favorecer a reusabilidade, a coesão e a independência de tecnologia
.
Por que importa: Permite que o sistema seja flexível e suporte manutenções frequentes sem "engessar" o desenvolvimento
.
Erros comuns: Achar que arquitetura é apenas sobre "onde colocar os arquivos" e esquecer que o objetivo é minimizar o esforço humano para manter o sistema
.
Exemplo prático: Registro de Usuário
Incorreto: Criar uma função que recebe o usuário, valida o e-mail, salva no banco e envia e-mail, tudo misturado com o código do Express.
Correto: Separar o que é "intenção" (cadastrar usuário) do que é "ferramenta" (Banco de dados/HTTP)
.
Comparação: No incorreto, se o banco mudar, você mexe em tudo. No correto, as regras centrais permanecem intactas
.
Aprendizado: A Arquitetura Limpa separa a Política (regras de negócio) do Mecanismo (tecnologia)
.

--------------------------------------------------------------------------------
2. Separação de Camadas
O que é: O sistema é dividido em círculos concêntricos. A regra de ouro é a Regra de Dependência: as dependências de código só podem apontar para dentro
.
Por que importa: Garante que as camadas centrais (estáveis) não saibam nada sobre as camadas externas (voláteis como UI e Banco)
.
Erros comuns: Importar o banco de dados ou um framework web dentro de uma regra de negócio
.
// INCORRETO: Camada interna dependendo de externa
import { db } from "./database"; // Erro! Regra interna conhecendo o banco.
function registerUser(data: any) {
  db.save(data); 
}

// CORRETO: Camada externa depende da interna
// O controlador (externo) chama o caso de uso (interno)
Comparação: O modelo correto protege o "coração" do seu software de mudanças externas
.
Aprendizado: Camadas internas são abstratas; camadas externas são detalhes concretos
.

--------------------------------------------------------------------------------
3. Entidades
O que é: Objetos que encapsulam as regras de negócio globais da empresa
. São as classes menos propensas a mudar
.
Por que importa: Elas são o "ouro" da sua aplicação; funcionariam mesmo se não houvesse computador
.
Erros comuns: Colocar lógica de persistência (como user.save()) dentro da entidade
.
// CORRETO: Entidade com regra de negócio pura
export class User {
  constructor(public readonly name: string, public readonly email: string) {
    if (!email.includes("@")) throw new Error("Email inválido"); // Regra da entidade
  }
}
Comparação: O exemplo correto foca apenas no que define um "Usuário" no seu negócio, sem saber se ele vai para um SQL ou NoSQL
.

--------------------------------------------------------------------------------
4. Casos de Uso (Interactors)
O que é: Camada que implementa as regras específicas da sua aplicação. Ela coordena o fluxo de dados para as entidades
.
Por que importa: É onde a "intenção" do sistema reside (ex: "Registrar novo usuário")
.
Erros comuns: Colocar detalhes de formato de saída (como JSON ou HTML) aqui
.
// CORRETO: Caso de Uso focado no fluxo
export class CreateUserUseCase {
  constructor(private userRepository: IUserRepository) {}
  async execute(input: UserDTO) {
    const user = new User(input.name, input.email);
    await this.userRepository.save(user);
  }
}
Comparação: O caso de uso usa uma interface (IUserRepository) em vez de uma classe concreta de banco, mantendo-se isolado
.

--------------------------------------------------------------------------------
5. Interfaces e Adapters
O que é: Tradutores que convertem dados do formato mais conveniente para as regras de negócio para o formato de sistemas externos
.
Por que importa: Permite que as regras internas falem com o mundo exterior sem conhecê-lo
.
Erros comuns: Passar objetos do banco de dados diretamente para o Caso de Uso
.
Aprendizado: Atravesse fronteiras usando apenas estruturas de dados simples (DTOs), nunca "linhas de tabela"
.

--------------------------------------------------------------------------------
6. Controllers
O que é: Adaptadores que recebem a entrada do usuário (como uma requisição HTTP) e a transformam em um modelo de requisição para o Caso de Uso
.
Por que importa: Isola as regras de negócio do protocolo de comunicação (Web, Console, etc.)
.
Erros comuns: Colocar validação de negócio complexa ou lógica de banco no controller
.
// CORRETO: Controller apenas direciona dados
export class UserController {
  async handle(req: Request, res: Response) {
    const dto = { name: req.body.name, email: req.body.email };
    await this.createUserUseCase.execute(dto);
    return res.status(201).send();
  }
}
Comparação: O controller correto é magro e atua apenas como uma ponte
.

--------------------------------------------------------------------------------
7. Repositórios (Gateways)
O que é: Interfaces que definem como as regras de negócio acessam os dados, ocultando o banco de dados real
.
Por que importa: O banco de dados é um detalhe (I/O) e deve ser tratado como um plug-in
.
Erros comuns: Deixar o SQL vazar para as camadas superiores
.
// Interface (Interna)
export interface IUserRepository {
  save(user: User): Promise<void>;
}

// Implementação (Externa)
export class SqlUserRepository implements IUserRepository {
  async save(user: User) { /* Código SQL aqui */ }
}
Comparação: No modelo correto, você pode trocar o SqlUserRepository por um MongoUserRepository sem tocar na regra de negócio
.

--------------------------------------------------------------------------------
8. Inversão de Dependência (DIP)
O que é: Princípio onde módulos de alto nível não dependem de módulos de baixo nível; ambos dependem de abstrações
.
Por que importa: É o que permite que o fluxo de controle vá de fora para dentro, mas a dependência de código vá de fora para dentro
.
Erros comuns: Usar new Database() dentro do seu Caso de Uso.
Aprendizado: Use injeção de dependência para passar as ferramentas externas para o seu núcleo
.

--------------------------------------------------------------------------------
9. Testabilidade e TDD
O que é: A capacidade de testar regras de negócio sem precisar de um banco de dados ou servidor web rodando
.
Por que importa: Testes dão a coragem necessária para refatorar e melhorar o código continuamente
.
TDD: Escrever o teste antes do código de produção para garantir um design desacoplado
.
Exemplo: Criar um teste que usa um repositório em memória para validar se o usuário é cadastrado corretamente. Se o teste passar no seu laptop a 30 mil pés de altura (sem internet), sua arquitetura está no caminho certo
.

--------------------------------------------------------------------------------
10. Erros Comuns e Considerações Finais
Casar-se com Frameworks: Herdar classes de frameworks dentro das entidades torna você "escravo" do autor da biblioteca
.
Ignorar o Valor da Estrutura: Focar apenas em fazer o código funcionar hoje, ignorando se ele será fácil de mudar amanhã
.
Rigidez Oficial: Quando ninguém mais tem coragem de tocar no código por medo de quebrá-lo
.

--------------------------------------------------------------------------------
Trilha de Estudo Sugerida
Fundamentos de Clean Code: Funções pequenas, nomes expressivos e evitar comentários desnecessários
.
Princípios SOLID: Foco especial em Responsabilidade Única (SRP) e Inversão de Dependência (DIP)
.
Padrão TDD: Aprenda as 3 leis e como o ciclo de feedback rápido melhora o design
.
Modelagem de Domínio: Entenda o problema de negócio antes de codificar (Entidades e Casos de Uso)
.
Estudo de Caso Prático: Implemente uma funcionalidade pequena (como o Cadastro de Usuários) aplicando o desacoplamento do banco e da web
