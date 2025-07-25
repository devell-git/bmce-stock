# Sistema de Controle e Movimentação de Estoque  

**Laboratório de Análises Clínicas (Multi-Tenant)**  
Tecnologia: Laravel 10 + Filament | PHP 8.x | MySQL | OpenAI Responses API

---

## 1. Visão Geral  
Sistema modular, auditável e extensível para gestão de estoque e movimentações de vários clientes (tenants).  
- **Multi-Tenant**: cada cliente tem seus próprios dados; apenas **units** e **movement_types** são globais.  
- **Importação Inteligente**: CSV/Excel em formatos variados, mapeamento automático via OpenAI Responses API, preview e carga definitiva.  
- **UX/UI**: interfaces Filament customizadas, responsivas, com Livewire para interatividade em tempo real.

---

## 2. Módulos e Funcionalidades

| Módulo                          | Funcionalidades Principais                                                                                         |
|---------------------------------|---------------------------------------------------------------------------------------------------------------------|
| **Autenticação & RBAC**         | • Login/Logout (Sanctum)<br>• Roles & Permissions (Spatie)<br>• Policies & Gates                                    |
| **Cadastro Mestre**             | • **Globais**: units, movement_types<br>• **Por Cliente**: manufacturers, brands, categories/subcategories, areas/subáreas |
| **Produtos**                    | • CRUD multi-tenant: SKU, nome, marca, unidade, categoria, estoque mínimo/máximo<br>• Associação Produtos⇄Áreas      |
| **Movimentações**               | • Entrada/Saída (lote, validade, fornecedor, NF, valor, PDF, barcode)<br>• Workflow: solicitação (pendente→aprova/rejeita)<br>• Destaque de faltas de estoque<br>• Filtros & exportação XLS/PDF |
| **Inventário Cíclico**          | • Contagem por área (sistema vs. contado)<br>• Importação CSV/Excel com AI<br>• Auditoria de discrepâncias          |
| **Importação Inteligente**      | • Upload, escolha de tabela destino<br>• Mapeamento AI (import_mappings)<br>• Preview, ajustes, carga batch        |
| **Configurações Gerais**        | • Chaves de API (OpenAI, Webhooks)<br>• Branding (logo, favicon)<br>• Tema (light/dark), cores (color picker)<br>• Parâmetros (alertas de validade, reorder point) |
| **Relatórios & Dashboards**     | • Validades a vencer<br>• Giro de estoque<br>• Discrepâncias<br>• Consumo por fornecedor<br>• KPIs & SLAs           |
| **Integrações**                 | • API RESTful v1 (Sanctum)<br>• Webhooks (eventos principais)<br>• Extensão para LIMS e scanners/barcode           |

---

## 3. Modelo de Dados Detalhado

### 3.1. Tabela `clients`
```sql
CREATE TABLE clients (
  id            BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  name          VARCHAR(255) NOT NULL,
  subdomain     VARCHAR(100) UNIQUE NULL,
  created_by    BIGINT UNSIGNED NOT NULL,
  created_at    TIMESTAMP NULL,
  updated_at    TIMESTAMP NULL
);
````

### 3.2. Entidades Compartilhadas (sem `client_id`)

```sql
CREATE TABLE units (
  id            BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  abbreviation  VARCHAR(10) NOT NULL,
  description   VARCHAR(255) NULL,
  created_at    TIMESTAMP NULL,
  updated_at    TIMESTAMP NULL
);

CREATE TABLE movement_types (
  id            BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  direction     ENUM('in','out') NOT NULL,
  description   VARCHAR(100) NOT NULL,
  created_at    TIMESTAMP NULL,
  updated_at    TIMESTAMP NULL
);
```

### 3.3. Entidades Multi-Tenant (`client_id` obrigatório)

```sql
-- Manufacturers
CREATE TABLE manufacturers (
  id            BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  client_id     BIGINT UNSIGNED NOT NULL,
  name          VARCHAR(255) NOT NULL,
  contact_info  TEXT NULL,
  created_at    TIMESTAMP NULL,
  updated_at    TIMESTAMP NULL,
  FOREIGN KEY (client_id) REFERENCES clients(id) ON DELETE CASCADE
);

-- Brands
CREATE TABLE brands (
  id            BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  client_id     BIGINT UNSIGNED NOT NULL,
  name          VARCHAR(255) NOT NULL,
  manufacturer_id BIGINT UNSIGNED NOT NULL,
  created_at    TIMESTAMP NULL,
  updated_at    TIMESTAMP NULL,
  FOREIGN KEY (client_id) REFERENCES clients(id) ON DELETE CASCADE,
  FOREIGN KEY (manufacturer_id) REFERENCES manufacturers(id)
);

-- Categories / Subcategories
CREATE TABLE categories (
  id            BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  client_id     BIGINT UNSIGNED NOT NULL,
  name          VARCHAR(255) NOT NULL,
  parent_id     BIGINT UNSIGNED NULL,
  created_at    TIMESTAMP NULL,
  updated_at    TIMESTAMP NULL,
  FOREIGN KEY (client_id) REFERENCES clients(id) ON DELETE CASCADE,
  FOREIGN KEY (parent_id) REFERENCES categories(id)
);

-- Areas / Subáreas
CREATE TABLE areas (
  id            BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  client_id     BIGINT UNSIGNED NOT NULL,
  name          VARCHAR(255) NOT NULL,
  parent_id     BIGINT UNSIGNED NULL,
  created_at    TIMESTAMP NULL,
  updated_at    TIMESTAMP NULL,
  FOREIGN KEY (client_id) REFERENCES clients(id) ON DELETE CASCADE,
  FOREIGN KEY (parent_id) REFERENCES areas(id)
);

-- Products
CREATE TABLE products (
  id            BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  client_id     BIGINT UNSIGNED NOT NULL,
  sku           VARCHAR(100) NOT NULL,
  name          VARCHAR(255) NOT NULL,
  brand_id      BIGINT UNSIGNED NOT NULL,
  unit_id       BIGINT UNSIGNED NOT NULL,
  category_id   BIGINT UNSIGNED NOT NULL,
  min_stock     DECIMAL(10,2) NULL,
  max_stock     DECIMAL(10,2) NULL,
  created_at    TIMESTAMP NULL,
  updated_at    TIMESTAMP NULL,
  FOREIGN KEY (client_id) REFERENCES clients(id) ON DELETE CASCADE,
  FOREIGN KEY (brand_id) REFERENCES brands(id),
  FOREIGN KEY (unit_id) REFERENCES units(id),
  FOREIGN KEY (category_id) REFERENCES categories(id),
  UNIQUE KEY client_sku_unique (client_id, sku)
);

-- Pivot Products ⇄ Areas
CREATE TABLE product_area (
  product_id    BIGINT UNSIGNED NOT NULL,
  area_id       BIGINT UNSIGNED NOT NULL,
  client_id     BIGINT UNSIGNED NOT NULL,
  PRIMARY KEY (product_id, area_id),
  FOREIGN KEY (product_id) REFERENCES products(id),
  FOREIGN KEY (area_id) REFERENCES areas(id),
  FOREIGN KEY (client_id) REFERENCES clients(id) ON DELETE CASCADE
);

-- Movements
CREATE TABLE movements (
  id              BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  client_id       BIGINT UNSIGNED NOT NULL,
  product_id      BIGINT UNSIGNED NOT NULL,
  area_id         BIGINT UNSIGNED NOT NULL,
  movement_type_id BIGINT UNSIGNED NOT NULL,
  supplier        VARCHAR(255) NULL,
  lot             VARCHAR(100) NULL,
  expiry_date     DATE NULL,
  quantity        DECIMAL(10,2) NOT NULL,
  unit_price      DECIMAL(12,4) NOT NULL,
  total_price     DECIMAL(14,4) GENERATED ALWAYS AS (quantity * unit_price) STORED,
  invoice_number  VARCHAR(100) NULL,
  movement_date   DATETIME NOT NULL,
  requester_id    BIGINT UNSIGNED NOT NULL,
  handler_id      BIGINT UNSIGNED NULL,
  status          ENUM('pending','approved','rejected') NOT NULL DEFAULT 'pending',
  notes           TEXT NULL,
  file_path       VARCHAR(255) NULL,
  barcode         VARCHAR(100) NULL,
  created_at      TIMESTAMP NULL,
  updated_at      TIMESTAMP NULL,
  FOREIGN KEY (client_id) REFERENCES clients(id) ON DELETE CASCADE,
  FOREIGN KEY (product_id) REFERENCES products(id),
  FOREIGN KEY (area_id) REFERENCES areas(id),
  FOREIGN KEY (movement_type_id) REFERENCES movement_types(id)
);

-- Inventory Counts
CREATE TABLE inventory_counts (
  id               BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  client_id        BIGINT UNSIGNED NOT NULL,
  area_id          BIGINT UNSIGNED NOT NULL,
  product_id       BIGINT UNSIGNED NOT NULL,
  system_qty       DECIMAL(10,2) NOT NULL,
  counted_qty      DECIMAL(10,2) NOT NULL,
  discrepancy      DECIMAL(10,2) GENERATED ALWAYS AS (counted_qty - system_qty) STORED,
  counted_at       DATETIME NOT NULL,
  user_id          BIGINT UNSIGNED NOT NULL,
  audit_log        JSON NULL,
  created_at       TIMESTAMP NULL,
  updated_at       TIMESTAMP NULL,
  FOREIGN KEY (client_id) REFERENCES clients(id) ON DELETE CASCADE,
  FOREIGN KEY (area_id) REFERENCES areas(id),
  FOREIGN KEY (product_id) REFERENCES products(id)
);

-- Users & Permissions
CREATE TABLE users (
  id            BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  client_id     BIGINT UNSIGNED NOT NULL,
  name          VARCHAR(255) NOT NULL,
  email         VARCHAR(255) UNIQUE NOT NULL,
  password      VARCHAR(255) NOT NULL,
  created_at    TIMESTAMP NULL,
  updated_at    TIMESTAMP NULL,
  FOREIGN KEY (client_id) REFERENCES clients(id) ON DELETE CASCADE
);
-- tabelas roles, permissions, model_has_roles, role_has_permissions conforme Spatie
```

---

## 4. Detalhamento de Telas e Componentes

### 4.1. Login & Perfil

* **LoginPage** (FilamentAuth): campos Email (Input), Senha (PasswordInput), Lembrar-me (Checkbox).
* **ProfilePage**: Avatar (FileUpload), Nome (TextInput), E-mail (TextInput readonly), Trocar Senha (PasswordInput).

### 4.2. Dashboard Principal

* **Cards KPI** (Filament Widgets)

  * Estoque abaixo do mínimo (Badge + número)
  * Movimentações pendentes (Badge, Link para ListMovementsPage)
  * Lotes vencendo em X dias (TableList com colunas produto/lote/data)
* **Charts** (Livewire + ChartJS)

  * Entradas x Saídas (line chart)
  * Consumo por área (bar chart)

### 4.3. CRUD Mestre (Resources Filament)

#### Manufacturers & Brands

* **List**: Table (Columns: Nome, Contato, Ações)
* **Form**:

  * Nome (TextInput required)
  * Contato (Textarea)
  * Manufacturer select (BelongsToSelect)

#### Units & Movement Types

* **List**: Table simples, ações inline (Edit/Delete)
* **Form**:

  * Sigla (TextInput)
  * Descrição (TextInput)
  * Direction (Toggle or Select in/out)

#### Categories & Areas

* **Tree List** (TreeList component)
* **Form**: Nome (TextInput), Pai (BelongsToSelect tree)

### 4.4. Products (Resource Filament)

* **List**:

  * Columns: SKU, Nome, Marca, Unidade, Categoria, Estoque Atual (computed), Ações
  * Filters: Marca, Categoria, Área (MultiSelectFilter)
  * Search: global por SKU/nome
* **Form**:

  * SKU (TextInput)
  * Nome (TextInput)
  * Marca (BelongsToSelect)
  * Unidade (BelongsToSelect)
  * Categoria (BelongsToSelect)
  * Estoque Mínimo / Máximo (NumericInput)
  * Áreas (MultiSelect)

### 4.5. Movements (Resource + Custom Pages)

#### ListMovementsPage

* **Table** (Livewire DataTable)

  * Colunas: SKU, Produto, Área, Tipo (Badge), Quantidade, Valor Total, Status (Badge colorido), Data, Ações
  * Filtros: Intervalo de Datas (DatePicker range), Tipo (Select), Status (Select), Produto (Select searchable), Área, Lote, Validade (DatePicker), Fornecedor (TextInput), Requester/Handler (Select)
  * Ações em massa: Export XLS, Export PDF, Marcar como Lido

#### CreateMovementPage / EditMovementPage

* **Form** em abas (Tabs component):

  1. **Dados Básicos**: Produto (BelongsToSelect + Autocomplete), Área, Tipo, Quantidade (NumericInput), Unidade (readonly)
  2. **NF & Valores**: Fornecedor (TextInput), Nº Nota (TextInput), Data (DatePicker), Valor Unitário (NumericInput), Valor Total (Computed Field)
  3. **Lote & Validade**: Lote (TextInput), Validade (DatePicker)
  4. **Anexos & Observações**: Upload NF (FileUpload PDF), Código de Barras (TextInput), Observações (Textarea)
* **Botões Flutuantes**: Salvar, Cancelar
* **Validations**: FormRequest com regras e mensagens claras.

#### Solicitation Flow

* **NewRequestPage**: tabela de produtos (DataTable) com colunas SKU, Nome, Estoque Atual, Quantidade Desejada (NumericInput inline) + Botão “Solicitar”.
* **RequestApprovalPage**: similar ao ListMovements, mas apenas pendentes → botões Aprovar/Rejeitar inline, com modal de comentário.

### 4.6. Inventário Cíclico

* **InventoryCountPage** (Custom Filament Page)

  * Select Área (Dropdown) → carrega tabela de produtos da área.
  * Table: colunas SKU, Nome, Estoque Sistema, Quantidade Contada (NumericInput inline), Discrepância (computed), Observações (Textarea inline)
  * Botões: “Importar CSV/Excel” (abre modal), “Salvar Contagem” (confirma e gera registro)

* **Import Modal**:

  * FileUpload (CSV/Excel)
  * Select Tabela Destino (categories: manufacturers, brands, products)
  * Botão “Processar via AI” → spinner Livewire
  * Após retorno AI: mostra preview (TablePreview component), e mapeamentos (Repeater: source\_column → target\_field, confidence)
  * Botão “Confirmar Importação”

### 4.7. Configurações Gerais (SettingsResource)

* **Tabs**:

  1. **Branding**: Logo (FileUpload), Favicon (FileUpload)
  2. **Tema**: Toggle Light/Dark, ColorPicker Primária, ColorPicker Secundária
  3. **APIs**: OpenAI Key (PasswordInput masked), Webhook URL (TextInput), Webhook Secret (TextInput)
  4. **Parâmetros**: Dias alerta validade (NumericInput), Ponto de Reorder (NumericInput)

---

## 5. Relatórios & Dashboards

### 5.1. Relatório: Validades a Vencer

* **Filtros**: Dias até o vencimento (Slider), Área, Categoria, Fornecedor
* **Output**: Table paginada com colunas: SKU, Produto, Lote, Validade, Dias Restantes, Quantidade
* **Export**: XLS / PDF

### 5.2. Relatório: Giro de Estoque

* **Filtros**: Período (DatePicker range), Produto, Área
* **Output**:

  * **Chart**: linhas de Entradas e Saídas ao longo do tempo
  * **Table**: SKU, Nome, Qtd Entradas, Qtd Saídas, Saldo Final
* **Export**: XLS / PDF

### 5.3. Relatório: Discrepâncias de Inventário

* **Filtros**: Contagem (select batch), Área
* **Output**: Table: SKU, Nome, Sistema, Contado, Discrepância, Responsável, Data Contagem
* **Export**: XLS / PDF

### 5.4. Relatório: Consumo por Fornecedor

* **Filtros**: Período, Fornecedor (multi-select)
* **Output**:

  * **Bar Chart**: Quantidade total por fornecedor
  * **Table**: Fornecedor, Qtd Total, Valor Total, % do Total
* **Export**: XLS / PDF

### 5.5. Dashboard de KPIs e SLAs

* **Widgets** (Filament Metrics):

  * Tempo médio de aprovação de solicitações (Gauge)
  * % de pedidos acima do estoque (Trend)
  * Aderência de inventário (%)
* **Gráficos** (ChartJS):

  * SLA de atendimento diário (linha)
  * Top 5 produtos mais movimentados (bar)

---

## 6. Plano de Tarefas (Epics → Stories → Tasks)

### Epic 1 – Infraestrutura & Tenancy

* Story 1.1: Setup Laravel + Filament + Spatie + Sanctum + Migrations de `clients`, `units`, `movement_types`.
* Story 1.2: Middleware Tenancy, ServiceLayer, Domain folders.

### Epic 2 – Cadastro Mestre

* Story 2.1: Manufacturers & Brands Resources
* Story 2.2: Categories & Areas (Tree)
* Story 2.3: Units & MovementTypes Globals

### Epic 3 – Produtos & Áreas

* Story 3.1: Products Resource + Validations
* Story 3.2: Pivot product\_area

### Epic 4 – Movimentações & Solicitações

* Story 4.1: Movements Resource, List/Edit/Create
* Story 4.2: NewRequestPage + Approval Flow
* Story 4.3: Jobs para estoque & notificações

### Epic 5 – Inventário & Importação AI

* Story 5.1: InventoryCountPage
* Story 5.2: Import Modal + AI Mapping Job
* Story 5.3: Preview & Import Batch Job

### Epic 6 – Configurações

* Story 6.1: SettingsResource (Tabs + Components)
* Story 6.2: Config Cache & UI Overrides

### Epic 7 – Relatórios & Dashboards

* Story 7.1: Report Pages + Exports
* Story 7.2: Dashboard Widgets & Charts

### Epic 8 – Integrações

* Story 8.1: API RESTful v1 (Resources + Rate Limit)
* Story 8.2: Webhooks (Listeners + HMAC)
* Story 8.3: Endpoints LIMS / Scanner

---

## 7. Boas Práticas & Padrões

* PSR-12, SOLID, DDD (onde fizer sentido)
* FormRequests centralizados
* Events & Listeners para auditoria/webhooks
* Jobs/Queues: importação, notificações
* Testes: Pest + Laravel Feature Tests
* Documentação: Swagger/OpenAPI (`l5-swagger`)
* CI/CD: pipeline de testes/deploy automático
