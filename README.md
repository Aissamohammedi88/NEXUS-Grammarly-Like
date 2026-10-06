# NEXUS-Grammarly-Like
      Alternative open source à Grammarly.     Frappe → correction affichée en terminal et dans le navigateur en moins d'une seconde.     Zéro dépendance — stdlib Python pure. Tourne sur Linux, macOS, Windows et a-Shell iOS.  Installation · Architecture · API · SEO Meta · Déploiement
{
  "point"      : "RULE_ID",
  "titre"      : "message humain",
  "explication": "description de la règle",
  "mot"        : "mot fautif",
  "offset"     : 42,
  "longueur"   : 5,
  "suggestions": ["correction1", "correction2"],
  "categorie"  : "GRAMMAR"
}

[HTML] — Frontend embarqué (SPA)

Interface complète injectée dans une string Python, servie par GET /.

Fonctionnement JS :

Keyup  → debounce 900ms
       → POST /api/corriger
       → parse response
       → render erreurs + texte corrigé

Polling 3s → GET /api/etat → màj barre de statut

Anti-rebond : derniereRequeteId incrémenté à chaque requête. Si une réponse plus récente arrive, la précédente est ignorée (évite les race conditions réseau).
[HANDLER] — Routeur HTTP
Méthode 	Route 	Description
GET 	/ 	Sert le frontend HTML
GET 	/api/etat 	État vivant (ping, compteurs, dernière correction)
GET 	/api/status 	Métadonnées du service
GET 	/api/ping 	Healthcheck simple
POST 	/api/corriger 	Correction d'un texte
POST 	/api/config/save 	Mise à jour de la config
OPTIONS 	* 	CORS preflight
[SERVER] — Serveur multi-thread

class Server(ThreadingMixIn, HTTPServer):
    daemon_threads   = True   # threads tués avec le process principal
    allow_reuse_address = True  # pas de TIME_WAIT sur restart

Chaque requête HTTP est traitée dans son propre thread. Thread-safety assurée par ETAT_LOCK et LOG_LOCK.
[MAIN] — Démarrage

main()
├── port_libre(HOST, PORT)       # bind test sans écoute
├── trouver_port(HOST, PORT+1)   # fallback auto si 8914 occupé
├── Thread(boucle_ping, daemon)  # ping LT toutes les 30s
└── Server.serve_forever()       # boucle principale

API REST
POST /api/corriger

Request

{
  "texte"  : "Il faut que je sois plus attentif.",
  "langue" : "fr",
  "auto"   : true
}

Response 200

{
  "ok"              : true,
  "backend"         : "languagetool",
  "endpoint"        : "https://api.languagetool.org/v2/check",
  "texte_original"  : "Il faut que je sois plus attentif.",
  "texte_corrige"   : "Il faut que je sois plus attentif.",
  "nb_corrections"  : 0,
  "corrections"     : [],
  "temps_ms"        : 312
}

Response 400

{ "ok": false, "err": "texte vide" }

GET /api/etat

{
  "ok": true,
  "etat": {
    "lt_ok"              : true,
    "lt_ms"              : 287,
    "lt_endpoint"        : "https://api.languagetool.org/v2/check",
    "derniere_correction": 1728233412.4,
    "dernier_nb"         : 3,
    "corrections_total"  : 47,
    "erreurs_total"      : 0
  }
}

POST /api/config/save

{
  "languagetool_url" : "http://localhost:8081/v2/check",
  "debounce_ms"      : 600,
  "langue_defaut"    : "en-US"
}

SEO & Meta tags

Le frontend HTML est embarqué dans la string HTML (ligne ~366).
Ajoute ces balises dans le <head> pour un score Lighthouse SEO 97–99 :

<!-- Identité -->
<meta name="description"
      content="NEXUS Grammarly-Like : correcteur orthographique temps réel en Python pur,
               sans dépendance externe. Interface web localhost + terminal coloré.
               Compatible a-Shell iOS. Propulsé par LanguageTool.">
<meta name="keywords"
      content="grammarly alternative, correcteur orthographique python, spell checker
               open source, languagetool python, real-time spell check, correction
               orthographe terminal, a-shell ios python, grammarly like open source,
               python spell check web, correcteur temps réel, stdlib python">
<meta name="author"       content="Aissa Mohammedi (DGK)">
<meta name="robots"       content="index, follow">
<meta name="revisit-after" content="7 days">
<meta name="language"     content="French, English">

<!-- Open Graph (partage social) -->
<meta property="og:type"        content="website">
<meta property="og:title"       content="NEXUS Grammarly-Like v4.0.0">
<meta property="og:description" content="Alternative open source à Grammarly.
  Python stdlib, zéro dépendance, correction à la frappe, iOS a-Shell.">
<meta property="og:image"       content="https://raw.githubusercontent.com/
  DGK-Aissa/nexus-grammarly-like/main/docs/preview.png">
<meta property="og:url"         content="http://localhost:8914/">
<meta property="og:site_name"   content="NEXUS Grammarly-Like">

<!-- Twitter Card -->
<meta name="twitter:card"        content="summary_large_image">
<meta name="twitter:title"       content="NEXUS Grammarly-Like — Python, zéro dépendance">
<meta name="twitter:description" content="Correcteur orthographique open source :
  frappe → correction en 900ms, web + terminal, a-Shell iOS.">
<meta name="twitter:image"       content="https://raw.githubusercontent.com/
  DGK-Aissa/nexus-grammarly-like/main/docs/preview.png">

<!-- Canonique -->
<link rel="canonical" href="http://localhost:8914/">

Checklist score 97-99 :
Critère 	Statut
<title> descriptif avec mot-clé principal 	✅ déjà présent
meta description 120–158 chars 	✅ à ajouter
meta keywords (15 termes ciblés) 	✅ à ajouter
og:image 1200×630px 	⚠️ créer docs/preview.png
lang="fr" sur <html> 	✅ déjà présent
<meta charset="utf-8"> 	✅ déjà présent
<meta name="viewport"> 	✅ déjà présent
Heading hierarchy (h1 → h2 → h3) 	✅ présent dans le HTML
<link rel="canonical"> 	✅ à ajouter
Ratio texte/code > 30% 	✅ le HTML embarqué contient du contenu

Pour exposer la page publiquement (score Lighthouse complet) : voir Déploiement.
Déploiement
Localhost (défaut)

python3 nexus_grammarly_live.py
# → http://localhost:8914

Derrière nginx (exposition LAN/WAN)

server {
    listen 443 ssl;
    server_name grammarly.example.com;

    ssl_certificate     /etc/ssl/certs/cert.pem;
    ssl_certificate_key /etc/ssl/private/key.pem;

    location / {
        proxy_pass         http://127.0.0.1:8914;
        proxy_set_header   Host $host;
        proxy_set_header   X-Real-IP $remote_addr;
        proxy_read_timeout 30s;
    }
}

systemd (service permanent)

# /etc/systemd/system/nexus-grammarly.service
[Unit]
Description=NEXUS Grammarly-Like
After=network.target

[Service]
Type=simple
User=ubuntu
ExecStart=/usr/bin/python3 /opt/nexus-grammarly-like/nexus_grammarly_live.py
Restart=on-failure
RestartSec=5
Environment=NO_COLOR=1

[Install]
WantedBy=multi-user.target

systemctl enable --now nexus-grammarly

GitHub Actions — CI

# .github/workflows/ci.yml
name: CI
on: [push, pull_request]
jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with: { python-version: '3.11' }
      - run: python3 -m py_compile nexus_grammarly_live.py
      - run: python3 -m flake8 nexus_grammarly_live.py --max-line-length 120 --ignore E501

Fichiers générés au runtime

~/Documents/nexus_grammarly_live/
├── config.json          ← config persistante (modifiable à chaud)
├── logs/
│   └── grammarly.log    ← log horodaté toutes les corrections
└── results/
    └── history.jsonl    ← JSONL : une correction par ligne

Format history.jsonl :

{"ts": "2026-10-06T14:23:11+00:00", "texte": "Il faut que je...", "nb_corrections": 2, "langue": "fr", "auto": true}
{"ts": "2026-10-06T14:24:05+00:00", "texte": "Bonjour le mond", "nb_corrections": 1, "langue": "fr", "auto": true}

Exploitable directement avec jq :

# Taux d'erreur moyen sur la session
jq '.nb_corrections' ~/Documents/nexus_grammarly_live/results/history.jsonl | awk '{s+=$1;n++} END{print s/n}'

Langues supportées

Toutes les langues supportées par LanguageTool. Exemples :
Code 	Langue
fr 	Français
en-US 	Anglais (US)
en-GB 	Anglais (UK)
de 	Allemand
es 	Espagnol
pt-BR 	Portugais (Brésil)
ar 	Arabe
auto 	Détection automatique

Passer la langue via l'API :

{ "texte": "Hello wrold", "langue": "en-US" }

Limites & considérations
Sujet 	Limite 	Contournement
LanguageTool cloud 	Rate limit API publique (~20 req/min) 	Déployer LT en local (JVM requise)
Taille texte 	2 MB max par requête 	Splitter le texte côté client
SSL 	CERT_NONE par défaut 	Désactivé pour compatibilité a-Shell. En prod : activer la vérification
Concurrence 	ThreadingMixIn (thread par requête) 	Pour charge élevée, passer à asyncio + aiohttp
Persistance 	JSONL non rotatif 	Ajouter rotation via logging.handlers.RotatingFileHandler
Contribuer

git clone https://github.com/DGK-Aissa/nexus-grammarly-like.git
git checkout -b feat/ma-feature

# Test syntaxe
python3 -m py_compile nexus_grammarly_live.py

# Convention commits
git commit -m "feat(lt): ajouter support de la langue auto-detect"
git commit -m "fix(html): corriger le debounce sur mobile Safari"
git commit -m "docs(api): documenter le schéma response /api/corriger"

Types : feat · fix · docs · refactor · perf · test · chore
Licence

NEXUS-OPEN-2.0 — Usage libre, modification libre, redistribution avec attribution.
Voir LICENSE.

Développé par Aissa Mohammedi (DGK) — 2026

Parce que la correction orthographique ne devrait pas coûter 30$/mois.
Python stdlib pure · Web + Terminal · a-Shell iOS  Version Python Stdlib iOS Licence LanguageTool 
