# 🎭 A História do Hacker: Explorando Acme Corp

> *"Cada sistema tem uma história para contar. Você só precisa saber como ouvir."* - Anônimo

---

## 🌙 O Início da Jornada

**Meia-noite. Silêncio na cidade.**

Alex, um hacker ético experiente, estava sentado em seu setup noturno. A tela do monitor iluminava seu rosto enquanto ele navegava pelos fóruns de segurança. Uma mensagem interessante havia aparecido:

*"Alguém já investigou a Acme Corp? Parece ter uma infraestrutura interessante..."*

Alex sorriu. Era exatamente o tipo de desafio que ele gostava. Uma empresa aparentemente normal, mas com "infraestrutura interessante" geralmente significava uma coisa: vulnerabilidades mal configuradas e sistemas legados esquecidos.

*"Vamos ver o que essa empresa está escondendo,"* pensou Alex, preparando seu ambiente de investigação.

### Preparando o Ambiente Docker

Alex sabia que o laboratório estava configurado com Docker. Ele verificou se os contêineres estavam rodando e acessou o ambiente Kali Linux.

```bash
# Carregar os contêineres
docker-compose up -d

# Verificando se os contêineres estão rodando
docker-compose ps

# Entrando no contêiner Kali Linux
docker exec -it kensei_kali /bin/bash

# Criando workspace organizado dentro do contêiner
mkdir -p /home/kali/investigations/acme-corp
cd /home/kali/investigations/acme-corp
```

---

## 🔍 Fase 1: O Reconhecimento Inicial

### "Sempre comece pelo básico"

Alex sabia que a primeira regra do reconhecimento era não pular etapas. Ele começou com o mais básico: descobrir todos os domínios e subdomínios associados à empresa.

# "Primeiro, vamos ver o que o Subfinder consegue descobrir sobre acme-corp-lab.com"
```bash
subfinder -d acme-corp-lab.com -o subdomains.txt
```

*"Interessante,"* Alex murmurou enquanto observava os resultados. O Subfinder havia encontrado quatro subdomínios:

- `www.acme-corp-lab.com`
- `admin.acme-corp-lab.com` 
- `dev.acme-corp-lab.com`
- `old.acme-corp-lab.com`

*"Agora isso é interessante. Um subdomínio 'old' geralmente significa sistemas legados. E sistemas legados geralmente significam... vulnerabilidades."*

### "Vamos tentar o Amass também, mas sem perder tempo"

Alex sabia que o Amass poderia encontrar subdomínios adicionais, mas também sabia que poderia ser muito lento.

```bash
# "Vamos tentar o Amass, mas com timeout para não perder tempo"
amass enum -passive -d acme-corp-lab.com -o amass_results.txt -v 
```

*"Como esperado, o Amass demorou muito. Mas não importa - o Subfinder já nos deu uma boa base para trabalhar."*

### "Sempre verifique se os sistemas estão realmente ativos"

Alex sabia que nem todos os subdomínios descobertos estariam ativos. Ele precisava verificar quais realmente respondiam.

```bash
# "Vamos ver quais desses subdomínios realmente existem"
cat subdomains.txt | dnsx -resp -silent -o resolved_subdomains.txt
```

Os resultados mostraram que todos os quatro subdomínios resolviam para IPs diferentes. *"Múltiplos IPs... isso pode significar uma infraestrutura distribuída ou balanceamento de carga."*

### "Agora vamos ver o que esses serviços realmente fazem"

```bash
# "Vamos sondar quais serviços web estão rodando"
cat subdomains.txt | httpx -title -tech-detect -status-code -o live_web_services.txt
```

Alex analisou os resultados:

- `www.acme-corp-lab.com` - S3 estático, normal para um site principal
- `admin.acme-corp-lab.com` - WordPress! *"Isso é interessante. Painéis administrativos WordPress são alvos frequentes."*
- `old.acme-corp-lab.com` - Retornou 301 (redirecionamento) *"Hmm, um redirecionamento. Vamos investigar isso."*

---

## 🎯 Fase 2: A Descoberta do Hub Central

### "Sempre investigue redirecionamentos"

O redirecionamento 301 no subdomínio `old` chamou a atenção de Alex. Ele sabia que redirecionamentos muitas vezes revelavam informações interessantes.

```bash
# "Vamos seguir esse redirecionamento e ver o que está por trás"
curl -L -s http://old.acme-corp-lab.com/ > legacy_page.html

# "Alternativamente, vamos acessar diretamente via IP para ser mais confiável"
curl -s http://3.94.82.59/ > legacy_page.html
```

*"Ahá!"* Alex exclamou quando viu o conteúdo da página. Era uma página legacy com links para outros serviços:

```html
<h1>Legacy System</h1>
<p>Sistema legado em operação desde 2015</p>
<ul>
<li><a href="/phpinfo.php">PHP Info</a></li>
<li><a href="http://admin.acme-corp-lab.com.s3-website-us-east-1.amazonaws.com">Admin Panel</a></li>
<li><a href="http://dev.acme-corp-lab.com.s3-website-us-east-1.amazonaws.com">API</a></li>
</ul>
```

*"Perfeito! Esta página legacy é um hub central. Ela conecta todos os outros serviços. E veja só - há comentários HTML com URLs S3 adicionais!"*

Alex rapidamente extraiu os comentários:

```bash
# "Vamos ver o que mais está escondido nos comentários"
awk '/^<!--/,/-->$/' legacy_page.html
```

*"Excelente! URLs S3 públicas com arquivos da empresa. Isso pode conter informações valiosas."*

### "Sempre verifique arquivos públicos em S3"

```bash
# "Vamos baixar esses arquivos S3"
curl --output company_info.txt -s https://acme-corp-lab-public-files-6nssymq7.s3.us-east-1.amazonaws.com/company_info.txt
curl --output employees.csv -s https://acme-corp-lab-public-files-6nssymq7.s3.us-east-1.amazonaws.com/employees.csv
```

*"Jackpot!"* Alex sorriu. Os arquivos continham informações detalhadas da empresa:

- Lista completa de funcionários com emails e telefones
- Informações sobre infraestrutura da empresa
- Endereços de escritórios
- Servidores de email e DNS

*"Esta empresa está vazando informações sensíveis. Em uma situação real, isso seria um problema sério de segurança."*

---

## 🔍 Fase 3: Explorando os Serviços Descobertos

### "Agora vamos testar esses links S3"

Alex sabia que os links S3 na página legacy provavelmente redirecionariam para IPs diretos. Ele testou cada um:

```bash
# "Vamos ver para onde esses links S3 redirecionam"
curl -I http://admin.acme-corp-lab.com.s3-website-us-east-1.amazonaws.com
curl -I http://dev.acme-corp-lab.com.s3-website-us-east-1.amazonaws.com
```

*"Como esperado! Redirecionamentos para IPs diretos. Isso é comum em infraestruturas AWS."*

### "Vamos acessar diretamente os IPs"

```bash
# "Acessando o serviço admin via IP direto"
curl -I http://54.152.245.201:80/

# "E o serviço dev"
curl -s http://34.207.53.34:3000/
```

O serviço admin retornou informações sobre WordPress, e o serviço dev mostrou uma API JSON com endpoints interessantes.

### "APIs são sempre alvos interessantes"

Alex testou os endpoints da API:

```bash
# "Vamos ver o que essa API pode nos contar"
curl -s http://34.207.53.34:3000/api/health
curl -s http://34.207.53.34:3000/api/system-info
```

*"Oh não..."* Alex suspirou quando viu o resultado do `/api/system-info`. A API estava expondo credenciais de banco de dados, chaves de API e informações sensíveis:

```json
{
  "database": {
    "host": "localhost",
    "user": "dev_user", 
    "password": "DevPass2024"
  },
  "api_keys": {
    "stripe": "sk_test_51AcmeCorp123456789",
    "aws": "AKIAIOSFODNN7EXAMPLE"
  }
}
```

*"Isso é um desastre de segurança. Em um ambiente de produção, essas credenciais poderiam ser usadas para comprometer toda a infraestrutura."*

---

## 🔍 Fase 4: Enumeração de Diretórios

### "Vamos ver o que mais está escondido no WordPress"

Alex decidiu fazer uma enumeração de diretórios no serviço admin para ver se havia outros caminhos interessantes:

```bash
# "Criando uma wordlist personalizada baseada no que já sabemos"
echo -e 'admin\napi\nbackup\nconfig\ndev\ngit\nlogin\nphpinfo\nphpmyadmin\ntest\nwww\nwp-admin\nwp-content\nwp-includes\nuploads\nfiles\nimages\ncss\njs\nassets' > custom_wordlist.txt

# "Vamos ver o que encontramos"
gobuster dir -u http://54.152.245.201:80 -w custom_wordlist.txt -o gobuster_results.txt -q
```

*"Interessante! Encontramos diretórios WordPress padrão. Isso confirma que é realmente um WordPress."*

---

## 🔍 Fase 5: Coletando IPs para Enumeração Ativa

### "OSINT me trouxe até aqui. Agora preciso dos IPs reais para o próximo passo."

Alex sabia que a investigação OSINT havia revelado informações valiosas, mas para realizar enumeração ativa de serviços (no Lab 2), ele precisaria dos endereços IP reais dos sistemas descobertos.

*"Vamos coletar todos os IPs que encontramos. Esses serão meus alvos para a próxima fase."*

#### 5.1: Descobrindo o IP por trás do Redirecionamento

Alex lembrou que `old.acme-corp-lab.com` tinha um redirecionamento HTTP. O domínio provavelmente resolve para um bucket S3, mas o redirecionamento aponta para o IP real do servidor.

```bash
# Seguir o redirecionamento e capturar o IP real
curl -I -L http://old.acme-corp-lab.com 2>&1 | grep -i "location"

# Ou usar curl verbose para ver todos os redirecionamentos
curl -v http://old.acme-corp-lab.com 2>&1 | grep -E "Connected to|Location:"
```

*"Aha! O redirecionamento aponta para um IP real. Esse é o servidor que está hospedando a página legacy, não o bucket S3."*

Alternativamente, Alex abriu o navegador e acessou `http://old.acme-corp-lab.com/`, depois inspecionou a URL final na barra de endereços após o redirecionamento.

*"Perfeito! Agora tenho o IP real do servidor legacy."*

#### 5.2: Coletando Todos os IPs Descobertos


**Por quê?** Organizar os IPs em um arquivo facilita o trabalho no Lab 2, onde cada IP será alvo de enumeração ativa (scan de portas, enumeração SMB, LDAP, etc.).

#### 5.3: Coletando IPs dos Links na Página Legacy

Alex acessou `http://old.acme-corp-lab.com/` no navegador e observou os links presentes na página. Alguns desses links apontavam para outros domínios ou diretamente para IPs.

```bash
# Baixar o HTML da página legacy seguindo redirecionamentos
curl -L -s http://old.acme-corp-lab.com/ > old_acme_page.html

# Extrair todos os links (href) da página
curl -v http://admin.acme-corp-lab.com.s3-website-us-east-1.amazonaws.com 2>&1 | grep -E "Connected to|Location:"
curl -v http://dev.acme-corp-lab.com.s3-website-us-east-1.amazonaws.com 2>&1 | grep -E "Connected to|Location:"
```

*"Interessante... a página legacy contém links para outros serviços. Vou testar cada link e seguir os redirecionamentos para descobrir os IPs reais."*

Alex organizou todos os IPs encontrados durante a investigação:

```bash
# Criar arquivo com lista de IPs para o Lab 2
cat > target_ips.txt << 'EOF'
# IPs descobertos na investigação OSINT - Lab 1
# Esses IPs serão usados para enumeração ativa no Lab 2

# IP do servidor old.acme-corp-lab.com (página legacy com links sensíveis)
<IP>

# IP do servidor dev.acme-corp-lab.com (API expondo credenciais)
<IP>

# IP do servidor admin.acme-corp-lab.com (WordPress administrativo)
<IP>
EOF
```

#### 5.4: Documentando os Alvos


*"Agora tenho uma lista clara de alvos. No Lab 2, vou enumerar cada um desses IPs para descobrir quais serviços estão rodando e como estão configurados."*

---

## 🔍 Fase 6: Análise com Ferramentas Avançadas

### "Vamos usar o SpiderFoot para uma análise mais profunda"

Alex sabia que o laboratório incluía o SpiderFoot rodando no Docker. Ele abriu o navegador e acessou `http://localhost:5001`.

*"Perfeito! O SpiderFoot está rodando no contêiner. Vou criar um novo scan para acme-corp-lab.com."*

Alex criou um novo scan no SpiderFoot, selecionando módulos relevantes para DNS, HTTP e outras fontes de dados.

*"O SpiderFoot vai coletar dados de fontes que não consegui acessar manualmente. É sempre bom ter múltiplas perspectivas."*

---

## 📊 Fase 7: Documentando os Achados

### "Sempre documente tudo. Conhecimento sem documentação é conhecimento perdido."

Alex começou a compilar um relatório detalhado de sua investigação:

```bash
# "Criando um relatório profissional"
cat > investigation_report.md << 'EOF'
# Relatório de Investigação OSINT - Acme Corp

## Resumo Executivo
Investigação revelou múltiplas vulnerabilidades críticas de segurança...

## Achados Principais
1. **Exposição de Credenciais**: API dev expõe credenciais de banco de dados
2. **Arquivos Públicos S3**: Informações sensíveis da empresa expostas
3. **Sistema Legacy**: Página legacy serve como hub central desprotegido
4. **WordPress Exposto**: Painel administrativo acessível publicamente

## Recomendações Críticas
1. Remover credenciais da API de desenvolvimento
2. Proteger arquivos S3 sensíveis
3. Implementar autenticação na página legacy
4. Revisar configurações de segurança do WordPress
EOF
```

---

## 🤔 Reflexões sobre a Mentalidade Hacker

### "O que aprendemos com esta investigação?"

Enquanto Alex finalizava seu relatório, ele refletiu sobre o processo:

**1. Curiosidade é a chave**
*"Sempre questione o que vê. Um redirecionamento 301 não é apenas um redirecionamento - pode ser uma porta para sistemas esquecidos."*

**2. Siga os links, mas pense como um investigador**
*"Cada link, cada arquivo, cada endpoint pode conter informações valiosas. A página legacy foi o ponto de entrada para tudo."*

**3. Use múltiplas ferramentas e perspectivas**
*"Ferramentas de linha de comando são poderosas, mas interfaces visuais como SpiderFoot e Neo4j oferecem insights diferentes. O ambiente Docker torna tudo mais organizado e reproduzível."*

**4. Documente tudo**
*"Em uma investigação real, você pode precisar voltar aos dados meses depois. Documentação clara é essencial."*

**5. Pense em termos de relacionamentos**
*"Não são apenas sistemas isolados - são redes de conexões. Entender essas conexões é fundamental."*

### "A mentalidade hacker não é sobre quebrar sistemas - é sobre entender como eles funcionam."

Alex fechou seu laptop e sorriu. Esta investigação havia revelado vulnerabilidades sérias, mas também havia demonstrado algo importante: como uma abordagem sistemática e metódica pode revelar informações valiosas sobre qualquer alvo.

*"Cada sistema conta uma história. Você só precisa saber como ouvir."*

---

## 🎯 Lições Aprendidas

### Para Futuros Investigadores:

1. **Comece sempre pelo básico** - Enumeração de subdomínios e DNS
2. **Configure seu ambiente adequadamente** - Docker torna tudo mais organizado e reproduzível
3. **Investigue redirecionamentos** - Eles podem revelar infraestrutura oculta
4. **Procure por arquivos públicos** - S3 buckets mal configurados são comuns
5. **Teste APIs** - Endpoints de desenvolvimento frequentemente expõem credenciais
6. **Use visualização** - Grafos ajudam a entender relacionamentos complexos
7. **Combine ferramentas CLI e web** - Interfaces visuais complementam comandos
8. **Documente tudo** - Conhecimento sem documentação é conhecimento perdido

### A Mentalidade Ética:

Alex sabia que as vulnerabilidades que havia encontrado eram sérias. Em um cenário real, ele teria:

1. **Reportado responsavelmente** as vulnerabilidades à empresa
2. **Fornecido evidências claras** de cada achado
3. **Sugerido correções específicas** para cada problema
4. **Respeitado prazos** para correção antes de divulgação pública

*"O verdadeiro hacker é aquele que usa suas habilidades para proteger, não para atacar."*

---

## 🌅 O Fim da Jornada... Por Enquanto

**5 da manhã. Primeira luz do dia.**

Alex finalizou seu relatório e o salvou. Ele havia descoberto vulnerabilidades significativas através de OSINT: subdomínios expostos, APIs vazando credenciais, buckets S3 mal configurados, e uma página legacy servindo como hub central desprotegido.

Mas algo o incomodava. Ele olhou para o arquivo `target_ips.txt` na tela.

*"OSINT me mostrou a superfície. Mas o que está rodando DENTRO desses servidores? SMB? LDAP? RDP? Que versões? Que configurações?"*

Alex sabia que a verdadeira investigação estava apenas começando. OSINT revelou **o quê** estava exposto. A próxima fase — enumeração ativa — revelaria **como** esses sistemas estavam configurados e **quão** vulneráveis realmente eram.

*"Esta empresa tem muito trabalho pela frente,"* pensou Alex, salvando o arquivo `target_ips.txt` com cuidado. *"Mas pelo menos agora tenho uma lista clara de alvos para a próxima fase."*

Ele configurou um lembrete para o dia seguinte:

```bash
# Salvar lista de tarefas para o Lab 2
cat > lab2_tasks.txt << 'EOF'
LAB 2 - ENUMERAÇÃO ATIVA DE SERVIÇOS

Alvos identificados no Lab 1:
- IP do servidor old.acme-corp-lab.com
- 34.207.53.34 (dev.acme-corp-lab.com)
- 54.152.245.201 (admin.acme-corp-lab.com)
- IP do servidor www.acme-corp-lab.com

Próximos passos:
1. Scan de portas completo em cada IP
2. Enumeração SMB (compartilhamentos, usuários, políticas)
3. Enumeração LDAP (estrutura do diretório, contas)
4. Análise de serviços web (headers, tecnologias, vulnerabilidades)
5. Banner grabbing para identificar versões exatas
6. Documentar tudo e propor mitigações

Autorização: ✅ Teste de penetração autorizado pela empresa
EOF
```

*"O ambiente Docker tornou tudo mais organizado e reproduzível. Amanhã será outro dia — e com alvos bem definidos, a investigação será ainda mais eficiente."*

Alex fechou o laptop e se preparou para dormir, mas sua mente já estava trabalhando nos próximos passos. Ele havia usado suas habilidades para o bem — descobrindo vulnerabilidades para que pudessem ser corrigidas, não exploradas.

**Fim da história... do Lab 1.**

**A jornada continua no Lab 2: Enumeração Ativa de Serviços.**

---

*Esta história é uma representação fictícia de técnicas OSINT éticas. Em investigações reais, sempre obtenha autorização adequada e siga as leis e regulamentações aplicáveis.*
