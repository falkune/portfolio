```yaml
security:

    # Gestion du hashage des mots de passe
    password_hashers:
        App\Entity\User:
            algorithm: auto # Symfony choisit le meilleur algorithme

    # Où et comment Symfony trouve les utilisateurs
    providers:
        app_user_provider:
            entity:
                class: App\Entity\User # Entité User
                property: email       # Identification par email

    # Définition des firewalls (zones de sécurité)
    firewalls:

        # Firewall pour les outils techniques et fichiers statiques
        dev:
            pattern: ^/(_profiler|_wdt|assets|build)/
            security: false # Aucune sécurité appliquée

        # Firewall principal de l'API
        api:
            pattern: ^/api
            stateless: true # Pas de session (API stateless)

            # Authentification via JSON
            json_login:
                check_path: /api/login_check # URL de login
                username_path: email         # Champ identifiant
                password_path: password      # Champ mot de passe

            # Authentification par JWT
            jwt: ~

    # Règles d'accès aux routes
    access_control:
        # Login accessible à tous
        - { path: ^/api/login_check$, roles: [PUBLIC_ACCESS] }

        # Route publique (ex: inscription)
        - { path: ^/api/user$, roles: [PUBLIC_ACCESS] }

        # Tout le reste de l'API nécessite une auth
        - { path: ^/api, roles: IS_AUTHENTICATED_FULLY }
```
