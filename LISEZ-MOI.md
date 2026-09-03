# Ammu-Nation — Caisse partagée (déploiement Vercel)

## Pourquoi ça affichait « registre injoignable »

La version précédente utilisait `window.storage`, une base fournie uniquement
par l'interface des artefacts Claude. Sur Vercel, cette API n'existe pas : la
page se charge, mais il n'y a plus rien derrière pour ranger les ventes.

Cette version détecte automatiquement où elle tourne :

| Contexte | Moteur utilisé | Partagé entre employés |
|---|---|---|
| Vercel avec Supabase configuré | Supabase | Oui |
| Vercel sans configuration | Navigateur (localStorage) | Non — indiqué dans l'app |
| Artefact Claude | `window.storage` | Oui |

Sans configuration, l'app fonctionne quand même et le dit clairement au lieu
d'afficher une erreur.

---

## Mettre en place le registre partagé (≈ 5 minutes, gratuit)

### 1. Créer le projet

Va sur [supabase.com](https://supabase.com), crée un compte, puis un nouveau
projet. Choisis une région proche (`East US` ou `Canada` fait l'affaire).

### 2. Créer les tables

Dans le menu de gauche : **SQL Editor** → **New query**. Colle ceci et clique
sur **Run**.

```sql
create table ammu_ventes (
  id          text primary key,
  date        timestamptz not null default now(),
  seller      text not null,
  client      text,
  pay         text,
  discount    int  default 0,
  items       jsonb not null default '[]',
  total       int  not null default 0,
  commission  int  not null default 0
);

create table ammu_config (
  id   int primary key,
  data jsonb not null
);

alter table ammu_ventes  enable row level security;
alter table ammu_config  enable row level security;

create policy "acces boutique ventes" on ammu_ventes
  for all using (true) with check (true);
create policy "acces boutique config" on ammu_config
  for all using (true) with check (true);

create index ammu_ventes_date_idx on ammu_ventes (date desc);
```

### 3. Récupérer les deux clés

Menu **Project Settings** → **API**. Note :

- **Project URL** — ressemble à `https://abcdefgh.supabase.co`
- **anon public** — une longue chaîne commençant par `eyJ...`

### 4. Les coller dans le fichier

Ouvre `index.html`, tout en haut du bloc `<script>` :

```js
const SUPABASE = {
  url: 'https://abcdefgh.supabase.co',
  key: 'eyJhbGciOi...'
};
```

### 5. Déployer

Pousse le dossier sur GitHub et importe-le dans Vercel, ou fais un
glisser-déposer du dossier sur [vercel.com/new](https://vercel.com/new).
Aucune configuration de build : c'est un site statique.

Le voyant en haut à droite doit afficher **Registre partagé** avec l'heure de
la dernière synchronisation.

---

## À savoir sur la sécurité

La clé `anon public` est faite pour être visible dans le code d'une page web,
mais avec la politique ci-dessus, **toute personne qui a l'adresse du site peut
lire, ajouter et supprimer des ventes**. C'est acceptable pour une armurerie RP
dont le lien circule en interne.

Si tu veux verrouiller davantage :

- garde l'URL Vercel privée et ne la partage que dans le Discord staff ;
- ou active l'authentification Supabase (Email / Discord) et remplace
  `using (true)` par `using (auth.role() = 'authenticated')` ;
- ou mets le site derrière la protection par mot de passe de Vercel
  (offre payante).

Ne mets jamais la clé `service_role` dans `index.html` : celle-là contourne
toutes les règles.

---

## Utilisation quotidienne

- **Vendeur** — l'employé entre son nom une fois, il est retenu sur son
  appareil. Les articles s'ajoutent au +/−, la remise est en pourcentage, la
  commission se calcule automatiquement.
- **Ventes des employés** — chiffre d'affaires, classement, historique
  dépliable. La liste se rafraîchit toutes les 45 secondes et au retour sur
  l'onglet, plus le bouton *Actualiser*.
- **Tarifs** — les prix et le taux de commission sont communs à l'équipe ; les
  ventes déjà passées gardent leur montant d'origine.
- **Export CSV** — exporte ce que les filtres affichent, ouvrable dans Excel ou
  Google Sheets pour la paie des commissions.
