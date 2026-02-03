**Résumé**
- **But**: Lier `User` et `Task` pour que chaque utilisateur publie des articles (tâches).

**Étapes (ordre recommandé)**
- **Modifier les entités**: ajouter une relation `ManyToOne` de `Task` vers `User` et une relation `OneToMany` de `User` vers `Task`.
- **Générer une migration Doctrine**: exécuter `php bin/console doctrine:migrations:diff` puis `php bin/console doctrine:migrations:migrate`.
- **Mettre à jour les fixtures**: attribuer un `User` comme auteur des `Task` de test.
- **Mettre à jour le controller**: lors de la création d'une `Task`, appeler `setAuthor($this->getUser())`.
- **Mettre à jour les templates / API**: afficher l'auteur (ex: email) dans les vues ou la réponse JSON.
- **Tester**: charger les fixtures, exécuter les migrations, vérifier en créant une tâche via l'UI/API.

**Fichiers à modifier (versions finales proposées)**

- `src/Entity/Task.php` (version finale suggérée)

```php
<?php

namespace App\Entity;

use App\Repository\TaskRepository;
use Doctrine\DBAL\Types\Types;
use Doctrine\ORM\Mapping as ORM;

#[ORM\Entity(repositoryClass: TaskRepository::class)]
#[ORM\Table(name: 'task')]
#[ORM\HasLifecycleCallbacks]
class Task
{
    public const STATUS_PENDING = 'pending';
    public const STATUS_IN_PROGRESS = 'in_progress';
    public const STATUS_COMPLETED = 'completed';

    #[ORM\Id]
    #[ORM\GeneratedValue]
    #[ORM\Column(type: Types::INTEGER)]
    private ?int $id = null;

    #[ORM\Column(type: Types::STRING, length: 255)]
    private ?string $title = null;

    #[ORM\Column(type: Types::TEXT, nullable: true)]
    private ?string $description = null;

    #[ORM\Column(type: Types::STRING, length: 20)]
    private string $status = self::STATUS_PENDING;

    #[ORM\Column(type: Types::DATETIME_MUTABLE)]
    private ?\DateTimeInterface $createdAt = null;

    #[ORM\Column(type: Types::DATETIME_MUTABLE, nullable: true)]
    private ?\DateTimeInterface $updatedAt = null;

    #[ORM\Column(type: Types::DATE_MUTABLE, nullable: true)]
    private ?\DateTimeInterface $dueDate = null;

    // Relation vers l'auteur (User)
    #[ORM\ManyToOne(targetEntity: User::class, inversedBy: 'tasks')]
    #[ORM\JoinColumn(nullable: false, onDelete: 'CASCADE')]
    private ?User $author = null;

    public function getId(): ?int
    {
        return $this->id;
    }

    public function getTitle(): ?string
    {
        return $this->title;
    }

    public function setTitle(string $title): static
    {
        $this->title = $title;
        return $this;
    }

    public function getDescription(): ?string
    {
        return $this->description;
    }

    public function setDescription(?string $description): static
    {
        $this->description = $description;
        return $this;
    }

    public function getStatus(): string
    {
        return $this->status;
    }

    public function setStatus(string $status): static
    {
        $validStatuses = [self::STATUS_PENDING, self::STATUS_IN_PROGRESS, self::STATUS_COMPLETED];
        if (!in_array($status, $validStatuses)) {
            throw new \InvalidArgumentException('Statut invalide');
        }
        $this->status = $status;
        return $this;
    }

    public function getCreatedAt(): ?\DateTimeInterface
    {
        return $this->createdAt;
    }

    public function setCreatedAt(\DateTimeInterface $createdAt): static
    {
        $this->createdAt = $createdAt;
        return $this;
    }

    public function getUpdatedAt(): ?\DateTimeInterface
    {
        return $this->updatedAt;
    }

    public function setUpdatedAt(?\DateTimeInterface $updatedAt): static
    {
        $this->updatedAt = $updatedAt;
        return $this;
    }

    public function getDueDate(): ?\DateTimeInterface
    {
        return $this->dueDate;
    }

    public function setDueDate(?\DateTimeInterface $dueDate): static
    {
        $this->dueDate = $dueDate;
        return $this;
    }

    public function getAuthor(): ?User
    {
        return $this->author;
    }

    public function setAuthor(User $author): static
    {
        $this->author = $author;
        return $this;
    }

    #[ORM\PrePersist]
    public function onPrePersist(): void
    {
        $this->createdAt = new \DateTime();
    }

    #[ORM\PreUpdate]
    public function onPreUpdate(): void
    {
        $this->updatedAt = new \DateTime();
    }

    public function toArray(): array
    {
        return [
            'id' => $this->id,
            'title' => $this->title,
            'description' => $this->description,
            'status' => $this->status,
            'createdAt' => $this->createdAt?->format('Y-m-d H:i:s'),
            'updatedAt' => $this->updatedAt?->format('Y-m-d H:i:s'),
            'dueDate' => $this->dueDate?->format('Y-m-d'),
            'author' => $this->author ? ['id' => $this->author->getId(), 'email' => $this->author->getEmail()] : null,
        ];
    }
}
```

- `src/Entity/User.php` (version finale suggérée)

```php
<?php

namespace App\Entity;

use App\Repository\UserRepository;
use Doctrine\Common\Collections\ArrayCollection;
use Doctrine\Common\Collections\Collection;
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

    #[ORM\OneToMany(mappedBy: 'author', targetEntity: Task::class)]
    private Collection $tasks;

    public function __construct()
    {
        $this->tasks = new ArrayCollection();
    }

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

    /** @return Collection|Task[] */
    public function getTasks(): Collection
    {
        return $this->tasks;
    }

    public function addTask(Task $task): static
    {
        if (!$this->tasks->contains($task)) {
            $this->tasks->add($task);
            $task->setAuthor($this);
        }
        return $this;
    }

    public function removeTask(Task $task): static
    {
        if ($this->tasks->removeElement($task)) {
            if ($task->getAuthor() === $this) {
                $task->setAuthor(null);
            }
        }
        return $this;
    }
}
```

- `src/DataFixtures/TaskFixtures.php` (extrait finalisé, création d'un user puis attribution)

```php
<?php

namespace App\DataFixtures;

use App\Entity\Task;
use App\Entity\User;
use Doctrine\Bundle\FixturesBundle\Fixture;
use Doctrine\Persistence\ObjectManager;

class TaskFixtures extends Fixture
{
    public function load(ObjectManager $manager): void
    {
        // Creer un utilisateur de test
        $user = new User();
        $user->setEmail('author@example.test');
        $user->setPassword('secret');
        $manager->persist($user);

        $tasksData = [
            ['title' => 'Exemple 1', 'description' => '...','status' => Task::STATUS_PENDING, 'createdAt' => 'now'],
            ['title' => 'Exemple 2', 'description' => '...','status' => Task::STATUS_PENDING, 'createdAt' => 'now'],
        ];

        foreach ($tasksData as $index => $taskData) {
            $task = new Task();
            $task->setTitle($taskData['title']);
            $task->setDescription($taskData['description']);
            $task->setStatus($taskData['status']);
            $task->setCreatedAt(new \DateTime($taskData['createdAt']));
            // Lier l'auteur
            $task->setAuthor($user);
            $manager->persist($task);
            $this->addReference('task_' . $index, $task);
        }

        $manager->flush();
    }
}
```

**Extrait de `TaskController` (création) — adapter la méthode de création pour fixer l'auteur**

```php
// Dans la méthode qui crée la Task (ex: store/post)
$task = new Task();
$task->setTitle($request->request->get('title'));
$task->setDescription($request->request->get('description'));
// ... autres champs ...
$task->setAuthor($this->getUser()); // <-- important
$entityManager->persist($task);
$entityManager->flush();
```

**Migration SQL suggérée**

```sql
ALTER TABLE task ADD author_id INT NOT NULL;
ALTER TABLE task ADD CONSTRAINT FK_task_author FOREIGN KEY (author_id) REFERENCES user (id) ON DELETE CASCADE;
```

Remarques et conseils rapides
- Avant de lancer la migration, assurez-vous qu'il n'y a pas de lignes dans `task` qui n'auraient pas d'auteur: si oui, soit rendre `author_id` nullable temporairement, soit attribuer des auteurs par script.
- Utilisez `php bin/console doctrine:migrations:diff` puis `php bin/console doctrine:migrations:migrate` pour générer/appliquer la migration proprement.
- Mettez à jour toutes les APIs ou templates qui créent `Task` pour renseigner l'auteur (généralement `setAuthor($this->getUser())`).

Fichiers mentionnés : [src/Entity/Task.php](src/Entity/Task.php), [src/Entity/User.php](src/Entity/User.php), [src/DataFixtures/TaskFixtures.php](src/DataFixtures/TaskFixtures.php), [src/Controller/TaskController.php](src/Controller/TaskController.php)

Si tu veux, je peux appliquer ces changements automatiquement (patcher les fichiers, générer la migration), ou seulement générer la migration et te fournir la commande à exécuter.
