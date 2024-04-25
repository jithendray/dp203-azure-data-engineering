## 1. Armazenamento de Dados:
### Azure Storage Account [Conta de Armazenamento Azure]

- Azure Blob Storage vs. Azure Data Lake Storage Gen2
- storage account -> access keys
- storage account -> shared access signature (SAS)
- storage account -> Redundancy
    - LRS (Armazenamento Localmente Redundante)
![triangle](https://upload.wikimedia.org/wikipedia/commons/thumb/e/ec/Regular_triangle.svg/255px-Regular_triangle.svg.png =100x)
    - ZRS (Armazenamento Redundante por Zonas)
	![rainbow triangle](https://images-wixmp-ed30a86b8c4ca887773594c2.wixmp.com/f/5ce56e4a-99f6-4328-bec6-49efa78b3c78/d4qq0ag-c02d7afb-215f-4440-ac5e-c1808978d0c7.gif?token=eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJzdWIiOiJ1cm46YXBwOjdlMGQxODg5ODIyNjQzNzNhNWYwZDQxNWVhMGQyNmUwIiwiaXNzIjoidXJuOmFwcDo3ZTBkMTg4OTgyMjY0MzczYTVmMGQ0MTVlYTBkMjZlMCIsIm9iaiI6W1t7InBhdGgiOiJcL2ZcLzVjZTU2ZTRhLTk5ZjYtNDMyOC1iZWM2LTQ5ZWZhNzhiM2M3OFwvZDRxcTBhZy1jMDJkN2FmYi0yMTVmLTQ0NDAtYWM1ZS1jMTgwODk3OGQwYzcuZ2lmIn1dXSwiYXVkIjpbInVybjpzZXJ2aWNlOmZpbGUuZG93bmxvYWQiXX0.2Uo1WDgBacqI7SsMBwbu47eqxQeNIlhEttIupysQXNA =100x)
    - GRS (Armazenamento Global Redundante)
![triangle](https://upload.wikimedia.org/wikipedia/commons/thumb/e/ec/Regular_triangle.svg/255px-Regular_triangle.svg.png =100x) ![globe](https://upload.wikimedia.org/wikipedia/commons/thumb/f/fa/Globe.svg/2048px-Globe.svg.png =100x)

    - GZRS (Armazenamento Global Redundante por Zonas)
![rainbow triangle](https://images-wixmp-ed30a86b8c4ca887773594c2.wixmp.com/f/5ce56e4a-99f6-4328-bec6-49efa78b3c78/d4qq0ag-c02d7afb-215f-4440-ac5e-c1808978d0c7.gif?token=eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJzdWIiOiJ1cm46YXBwOjdlMGQxODg5ODIyNjQzNzNhNWYwZDQxNWVhMGQyNmUwIiwiaXNzIjoidXJuOmFwcDo3ZTBkMTg4OTgyMjY0MzczYTVmMGQ0MTVlYTBkMjZlMCIsIm9iaiI6W1t7InBhdGgiOiJcL2ZcLzVjZTU2ZTRhLTk5ZjYtNDMyOC1iZWM2LTQ5ZWZhNzhiM2M3OFwvZDRxcTBhZy1jMDJkN2FmYi0yMTVmLTQ0NDAtYWM1ZS1jMTgwODk3OGQwYzcuZ2lmIn1dXSwiYXVkIjpbInVybjpzZXJ2aWNlOmZpbGUuZG93bmxvYWQiXX0.2Uo1WDgBacqI7SsMBwbu47eqxQeNIlhEttIupysQXNA =100x) ![globe](https://upload.wikimedia.org/wikipedia/commons/thumb/f/fa/Globe.svg/2048px-Globe.svg.png =100x)
    
#### Semelhanças e Diferenças:
As formas de redundância para o Armazenamento de Dados nas 	Contas de Armazenamento da Azure são muito semelhantes, com alguns pequenos  detalhas que as diferenciam. Vamos começar pelas semelhanças

Em todas as formas de redundância, os dados são copiados pelo menos 3 vezes. Por isso o formato triangular escolhido para os ícones

Na notação do triângulo preto e branco, os dados são copiados 3 (três) vezes para um mesmo local físico. No caso do triângulo com um globo (GRS), depois da cópia para local físico acontece uma cópia para uma região secundária escolhida pelo usuário
Exemplo:
 - Primário: (América do Sul) Sul do Brasil
	 - 3 copias locais
 - Secundário: (América do Norte) Canadá Central
	 - 3 copias em região secundária (através do LRS)

Independente da forma de redundância dos dados escolhida para o projeto, a Azure mantêm um alto padrão para a consistência desses dados, O LRS fornece pelo menos 99,9999999999% (11 noves) durabilidade de objetos durante um determinado ano.

Isso significa que a redundância localmente redundante (LRS) já é o suficiente para manter uma estabilidade dos dados para a maioria dos casos

Agora às diferenças
Os triângulos com arco-íris são utilizados para notar a presença de Zonas de Disponibilidade. Nesses casos (ZRS e GZRS) os dados não são replicados para uma mesma máquina física mas sim para diferentes máquinas em zonas de disponibilidade diferentes
Exemplo:
- ZRS
	- Zona de lançamento: (América do Sul) Sul do Brasil
	- Zona de copia2: (Ásia-Pacífico) Sudeste Asiático
	-   (Ásia-Pacífico) Coreia Central

Esse tipo de solução pode aumentar a velocidade de acesso de diferentes usuários espalhados pelas diferentes regiões do globo

É importante considerar as necessidades do projeto na hora de escolher a redundância dos dados

### Contas de Armazenamento -> Liga de Acesso
   - Quente: Dados frequentemente acessados
    - Frio: Dados acessados com menos frequência (3 a 6 meses)
    - Arquivo: Dados acessados com pouquíssima frequência (a mais de um ano)

### Contas de Armazenamento -> Manutenção de Ciclo de Vida

O serviço de Lifecycle Management da Azure Storage Accounts é uma funcionalidade que ajuda a gerenciar os dados armazenados na sua conta de armazenamento da Azure, permitindo que você automatize a transferência, exclusão ou arquivamento de dados mais antigos com base em regras definidas por você.

Essas regras podem ser baseadas em datas específicas, idade dos dados, tipo de arquivo, tamanho do arquivo, entre outros critérios.

Por exemplo, você pode configurar o Lifecycle Management para excluir automaticamente dados que tenham mais de 90 dias, mover dados para armazenamento frio após 30 dias, ou ainda arquivar dados após um ano.

O objetivo principal do serviço é ajudar a manter os custos de armazenamento baixos e otimizar a performance da conta de armazenamento, permitindo que você se concentre no gerenciamento de dados mais importantes e relevantes para o seu negócio.
