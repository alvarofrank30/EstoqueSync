# EstoqueSync · Distribuidora Rio Capim Bebidas (empresa fictícia)

Sistema de estoque integrado a pedidos, feito para a **2ª Avaliação de Projeto de Banco de Dados**.

**Problema (hipotético):** os vendedores lançam pedidos em um sistema que não enxerga o estoque, e acabam vendendo
produtos que não existem na prateleira. **Solução:** estoque e pedidos no **mesmo banco PostgreSQL**. O pedido só é
aceito se houver saldo, o saldo é atualizado por gatilhos, e outros sistemas consultam a disponibilidade por uma
view com perfil de acesso próprio (`sistema_vendas`).

Stack: Apache + PHP 8 (PDO) + PostgreSQL 16. Todos os dados são fictícios.

## Estrutura

```
sql/
  01_ddl.sql                criação das 10 tabelas e restrições
  02_seed.sql               povoamento (30 produtos, 30 clientes, 100 pedidos...)
  03_views.sql              4 views
  04_functions_triggers.sql 2 funções PL/pgSQL + gatilhos de auditoria e de regra de negócio
  05_indices.sql            4 índices (com justificativa)
  06_seguranca.sql          perfis (roles), GRANT e REVOKE
  07_consultas.sql          as 10 perguntas do negócio (+1 conferência)
app/                        interface web em PHP
backup/estoque.dump         backup (pg_dump -Fc)
docs/documentacao.pdf       dossiê completo do banco de dados
docs/der-conceitual.png     modelo conceitual (notação de Chen)
docs/modelo-logico.png      modelo lógico relacional (PostgreSQL 16)
docs/explain_analyze.md     evidência dos índices (antes e depois)
```

## Instalação (Ubuntu/Debian)

```bash
# 1. Pacotes
sudo apt install apache2 php libapache2-mod-php php-pgsql postgresql

# 2. Banco vazio + os 7 scripts, NESTA ORDEM, parando no primeiro erro
sudo -u postgres createdb estoque
for f in 01_ddl 02_seed 03_views 04_functions_triggers 05_indices 06_seguranca 07_consultas; do
  sudo -u postgres psql -v ON_ERROR_STOP=1 -d estoque -f sql/$f.sql || break
done

# 3. Troque as senhas provisórias dos perfis (use a mesma no .env)
sudo -u postgres psql -d estoque -c "ALTER ROLE app_sistema PASSWORD 'SUA_SENHA'"
sudo -u postgres psql -d estoque -c "ALTER ROLE administrador PASSWORD 'OUTRA_SENHA'"
sudo -u postgres psql -d estoque -c "ALTER ROLE sistema_vendas PASSWORD 'MAIS_UMA'"

# 4. Publicar a aplicação
sudo cp -r app /var/www/html/estoque
sudo cp /var/www/html/estoque/.env.example /var/www/html/estoque/.env
sudo nano /var/www/html/estoque/.env          # DB_PASS = a senha do app_sistema
sudo chown -R www-data:www-data /var/www/html/estoque

# 5. Liberar o .htaccess (protege o .env) e reiniciar o Apache
#    Em /etc/apache2/conf-available/estoque.conf:
#      <Directory /var/www/html/estoque>
#          AllowOverride All
#      </Directory>
sudo a2enconf estoque && sudo systemctl restart apache2

# 6. Conferir
#    http://localhost/estoque/              -> a aplicação
#    http://localhost/estoque/.env          -> DEVE responder 403 (Forbidden)
```

O arquivo `.env` **nunca** vai para o Git (já está no `.gitignore`).

## Backup e restauração

```bash
# Gerar
sudo -u postgres pg_dump -Fc --no-owner --no-acl -d estoque -f backup/estoque.dump

# Restaurar num banco novo (perfis: rodar o 06 depois, ele reaplica os GRANTs)
sudo -u postgres createdb estoque_restaurado
sudo -u postgres pg_restore --no-owner -d estoque_restaurado backup/estoque.dump
sudo -u postgres psql -v ON_ERROR_STOP=1 -d estoque_restaurado -f sql/06_seguranca.sql
```

## Roteiro da demonstração (5 passos, 3 minutos)

1. **O problema:** em *Entrada de estoque* anote o saldo de um produto; em *Novo pedido* tente vender mais que o saldo.
   O sistema recusa e informa **"O pedido NÃO foi gravado"** (é o ROLLBACK da transação).
2. **O fluxo feliz:** lance um pedido com 2 itens e abra o detalhe. No terminal:
   `SELECT * FROM estoque WHERE id_produto = X;` mostra o saldo já descontado (gatilho `trg_baixar_estoque`).
3. **O estorno:** em *Pedidos*, cancele o pedido. O saldo volta, e `SELECT * FROM movimentacao_estoque ORDER BY 1 DESC LIMIT 5;` mostra o `estorno`.
4. **A integração entre sistemas:** conecte como `sistema_vendas` (`psql -U sistema_vendas -h localhost -d estoque`),
   consulte `SELECT * FROM vw_saldo_produto;` e tente `SELECT * FROM produto;` (negado).
5. **A auditoria:** `SELECT tabela, operacao, usuario_bd, usuario_app, momento FROM auditoria ORDER BY 1 DESC LIMIT 5;`
   (como administrador): aparece o nome do vendedor que operou a tela.

## Regras de negócio e onde são garantidas

| RN | Regra | Garantida por |
|---|---|---|
| RN01 | Não se vende mais do que o saldo | `fn_aplicar_movimentacao` (EST01) |
| RN02 | Cada item vendido baixa o estoque automaticamente | `trg_baixar_estoque` → `trg_aplicar_movimentacao` |
| RN03 | Cancelar o pedido devolve os itens ao estoque | `trg_estornar_pedido` |
| RN04 | O saldo nunca fica negativo | `ck_estoque_saldo` |
| RN05 | Produto, cliente e fornecedor são desativados, nunca apagados | coluna `ativo` + ausência de DELETE para `app_sistema` |
| RN06 | Alerta de produto abaixo do mínimo | `vw_estoque_critico` |
| RN07 | O preço é congelado no item (o catálogo pode mudar depois) | `fn_preparar_item_pedido` + `preco_unitario` |
| RN08 | Produto desativado não pode ser vendido | `fn_preparar_item_pedido` (EST02) |
| RN09 | Pedido cancelado é definitivo; pedido entregue não cancela | `trg_validar_status_pedido` (EST05, EST06) |
| RN10 | Itens e movimentações não se editam nem se apagam (cancela-se o pedido) | `trg_bloquear_*` (EST07) |
| RN11 | Todo cadastro e todo pedido são auditados (quem, o quê, quando) | `fn_auditar` |
| RN12 | A aplicação não usa superusuário e não altera saldo diretamente | perfil `app_sistema`, `GRANT UPDATE (estoque_minimo)` |
| RN13 | Saldo = entradas + estornos − saídas | conferência `C11` em `07_consultas.sql` |
