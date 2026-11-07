Application d'organisation des JOJ Dakar 2026 : une API NestJS (`apps/api`), un front React (`apps/web`) et des schémas partagés (`packages/shared`), dans un monorepo pnpm. Commandes et structure détaillée : `README.md`.

## Méthodologie

- Toute fonctionnalité suit `/prd` → `/spec` → `/spec-plan` → `/tdd-task` → `/converge` ; règles communes dans `specs/constitution.md`.
- Test d'abord, vu en échec par `pnpm tdd:red <fichier>`, puis code minimal jusqu'au vert.
- Chaque scénario d'acceptation a un test dont le titre commence par son identifiant : `it('001-AS2: rejects a full slot', …)` ; `pnpm spec:check` le vérifie.
- Une tâche = un commit Conventional Commits qui cite tâche et scénario : `feat(benevoles): T03 refuse a full slot (001-AS2)`. Jamais de `--no-verify`.
- Celui qui produit ne valide pas : le sous-agent `relecteur` relit dans un contexte vierge avant `/converge`.
- Le dev tourne dans Docker (install, build, test, dev) pour éviter les conflits de dépendances sur l'hôte.

## Architecture

- Un module par domaine dans `apps/api/src/modules/<domaine>/`, découpé en `domain/`, `application/`, `infra/`, `http/` ; modèle : `disciplines`.
- `domain/` reste en TypeScript pur, sans import de NestJS ni de Drizzle (vérifié par le lint).
- `application/` expose un port (classe abstraite `*.repository.ts`), un cas d'usage par classe (une méthode `execute()`) et un repository en mémoire dans `testing/`.
- `infra/` contient la table Drizzle (`*.schema.ts`), un mapper `toDomain()` / `toRow()` et le repository Drizzle, testé en intégration sur PGlite (`*.int.spec.ts`).
- `http/` : controller fin qui valide via DTO Zod (`createZodDto`), appelle un cas d'usage, sérialise avec `@ZodSerializerDto`.
- `*.module.ts` relie le port à son implémentation Drizzle ; ajouté aux `imports` de `src/app.module.ts`.
- Transverse dans `src/core/` : config, `AppError`, `ProblemDetailsFilter` (RFC 9457), base, santé.
- Schémas d'entrée et de réponse Zod dans `packages/shared/src/<nom>.ts`, partagés par l'API et le front (y compris la fausse API MSW).
- Front dans `apps/web/src/features/<nom>/` : `api/` (appels HTTP + hooks TanStack Query), `components/`, puis la route dans `src/routes/`.

## Concepts appréciés

- Typage strict, pas de `any` ; fonctions courtes.
- Une classe par cas d'usage, méthode unique `execute()`.
- Objets du domaine créés par `Discipline.create()` (applique les règles) et relus par `Discipline.restore()` (sans rejouer les règles).
- Le domaine ne fait pas confiance à la validation HTTP : il revalide (`validName`, `validCategory`).
- Controllers fins : aucune règle métier, seulement DTO Zod → cas d'usage → sérialisation.
- Codes d'erreur stables groupés dans `<nom>.errors.ts` avec leur catégorie (`VALIDATION`, `NOT_FOUND`, `CONFLICT`, `BUSINESS_RULE`), traduits en HTTP par le seul `ProblemDetailsFilter`.

## Bonnes pratiques

- Toute erreur métier est une `AppError` avec un code stable en majuscules ; jamais de `throw new HttpException` ni de `throw` d'une chaîne.
- Une entrée est validée par un schéma Zod de `packages/shared`, jamais à la main dans le controller.
- Pas de secret dans le code : les valeurs sensibles vont dans `apps/api/.env` (ignoré par git). Les hooks refusent les écritures qui ressemblent à un secret.
- Tests unitaires du domaine et de l'application avec le repository en mémoire ; intégration du repository Drizzle sur PGlite ; e2e de l'API via supertest ; front sous MSW avec la même fausse API pour tous les tests.
- Un audit d'accessibilité (`axeFindings(await runAxe())`) par état d'écran.
- `pnpm verify` (format, lint, types, tests liés aux fichiers changés) avant chaque commit ; lefthook l'exécute automatiquement.

## Nommage

- Fichiers en kebab-case : `create-discipline.ts`, `drizzle-discipline.repository.ts`, `disciplines.controller.ts`.
- Classes en PascalCase ; cas d'usage nommés par un verbe : `CreateDiscipline`, `ListDisciplines`.
- Codes d'erreur en majuscules avec préfixe du domaine : `DISCIPLINE_NAME_TAKEN`, `DISCIPLINE_NOT_FOUND`.
- Tests côté API : `*.spec.ts` (unitaires), `*.int.spec.ts` (intégration), `*.e2e.spec.ts` (bout en bout). Côté front : `<fichier>.test.tsx` à côté du code, `*.a11y.test.tsx` pour l'audit.
- Titres de tests en anglais, préfixés par le scénario : `it('001-AS2: rejects a full slot', …)`.
- Commits Conventional Commits avec tâche et scénario : `feat(disciplines): T02 create a discipline (000-AS1)`.
