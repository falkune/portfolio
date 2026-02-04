### etape pour la securisation des api:
pour securiser les api installer:
```sh
composer require lexik/jwt-authentication-bundle
```

## Solution 1 : Créer le Contrôleur d'Authentification

**Fichier** : `src/Controller/AuthController.php`

```php
<?php

namespace App\Controller;

use Lexik\Bundle\JWTAuthenticationBundle\Services\JWTTokenManagerInterface;
use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
use Symfony\Component\HttpFoundation\JsonResponse;
use Symfony\Component\Routing\Attribute\Route;

final class AuthController extends AbstractController
{
    #[Route('/api/login_check', name: 'login_check', methods: ['POST'])]
    public function login(JWTTokenManagerInterface $jwtManager): JsonResponse
    {
        $user = $this->getUser();

        if (!$user) {
            return $this->json(['error' => 'Invalid credentials'], 401);
        }

        $token = $jwtManager->create($user);

        return $this->json(['token' => $token]);
    }
}
```

**Explication** :
- La route `#[Route('/api/login_check', ...)]` définit le endpoint pour la connexion
- Le `JsonLoginAuthenticator` (du firewall) authentifie l'utilisateur automatiquement
- Si l'authentification réussit, `$this->getUser()` retourne l'utilisateur
- `$jwtManager->create($user)` génère le JWT
- La réponse retourne le token en JSON


modifier UserController
```php
<?php

namespace App\Controller;

use App\Entity\User;
use Doctrine\ORM\EntityManagerInterface;
use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
use Symfony\Component\HttpFoundation\Request;
use Symfony\Component\HttpFoundation\Response;
use Symfony\Component\HttpFoundation\JsonResponse;
use Symfony\Component\Routing\Attribute\Route;

final class UserController extends AbstractController
{
  #[Route('/api/user', name: 'api_user_register', methods: ['POST'])]
  public function register(Request $request, EntityManagerInterface $em): JsonResponse
  {
    $data = json_decode($request->getContent(), true);

    if (!$data || empty($data['email']) || empty($data['password'])) {
      return $this->json([
        'success' => false,
        'message' => 'Email et mot de passe requis!'
      ], Response::HTTP_BAD_REQUEST);
    }

    $user = new User();
    $user->setEmail($data['email']);
    $user->setPassword($data['password']); // hashé dans l'entité

    $em->persist($user);
    $em->flush();

    return $this->json([
      'success' => true,
      'message' => 'Utilisateur créé avec succès!'
    ], Response::HTTP_CREATED);
  }
}
```

---


## Solution 2 : Configurer le Firewall et les Routes Publiques
```yaml
security:
    password_hashers:
        App\Entity\User:
            algorithm: auto

    providers:
        app_user_provider:
            entity:
                class: App\Entity\User
                property: email

    firewalls:
        dev:
            pattern: ^/(_profiler|_wdt|assets|build)/
            security: false

        api:
            pattern: ^/api
            stateless: true
            json_login:
                check_path: /api/login_check
                username_path: email
                password_path: password
            jwt: ~

    access_control:
        - { path: ^/api/login_check$, roles: [PUBLIC_ACCESS] }
        - { path: ^/api/user$,       roles: [PUBLIC_ACCESS] }
        - { path: ^/api,             roles: IS_AUTHENTICATED_FULLY }
```


## Solution 3 : Générer les Clés RSA
```bash
mkdir -p config/jwt
openssl genrsa -out config/jwt/private.pem 4096
openssl rsa -pubout -in config/jwt/private.pem -out config/jwt/public.pem
```

**Explication** :
- **private.pem** : Clé privée (signe les tokens JWT)
- **public.pem** : Clé publique (vérifie les tokens JWT)
- **RSA 4096** : Algorithme et force de la clé
- **Non chiffrée** : Pas de passphrase (mode développement)

**Fichiers générés** :
```
config/jwt/
├── private.pem  (3272 bytes)
└── public.pem   (800 bytes)
```

**Configuration** ([.env](../.env)) :
```bash
JWT_SECRET_KEY=%kernel.project_dir%/config/jwt/private.pem
JWT_PUBLIC_KEY=%kernel.project_dir%/config/jwt/public.pem
JWT_PASSPHRASE=""  # Vide pour les clés non chiffrées
```

dans lexik_jwt_authentication.yaml on va retrouver 

```yml
lexik_jwt_authentication:
    secret_key: '%env(resolve:JWT_SECRET_KEY)%'
    public_key: '%env(resolve:JWT_PUBLIC_KEY)%'
    pass_phrase: '%env(JWT_PASSPHRASE)%'
    token_ttl: 3600 # pet ne pas y etre
```

si il le faut modifier l'entiter User:
```php
<?php

namespace App\Entity;

use App\Repository\UserRepository;
use Doctrine\ORM\Mapping as ORM;
use Symfony\Component\Security\Core\User\UserInterface;
use Symfony\Component\Security\Core\User\PasswordAuthenticatedUserInterface;

#[ORM\Entity(repositoryClass: UserRepository::class)]
class User implements UserInterface, PasswordAuthenticatedUserInterface
{
  #[ORM\Id]
  #[ORM\GeneratedValue]
  #[ORM\Column]
  private ?int $id = null;

  #[ORM\Column(length: 255, unique: true)]
  private ?string $email = null;

  #[ORM\Column(length: 255)]
  private ?string $password = null;

  public function getId(): ?int
  {
    return $this->id;
  }
  public function getEmail(): ?string
  {
    return $this->email;
  }
  public function setEmail(string $email): static
  {
    $this->email = $email;
    return $this;
  }
  public function getPassword(): ?string
  {
    return $this->password;
  }
  public function setPassword(string $password): static
  {
    $this->password = password_hash($password, PASSWORD_BCRYPT);
    return $this;
  }

  public function getRoles(): array
  {
    return ['ROLE_USER'];
  }
  public function getUserIdentifier(): string
  {
    return $this->email;
  }
  public function eraseCredentials(): void {}
  public function getSalt(): ?string
  {
    return null;
  }
}
```

