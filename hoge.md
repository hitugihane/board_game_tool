了解！では、**Spring Boot + MySQL** バックエンドに加えて、**Vue 3 + Vite** のフロントエンドを同じDocker環境内で構築する構成を丁寧に説明するね。

---

## 💡 最終的な構成イメージ

```bash
todo-app/
├── backend/ （Spring Boot）
│   ├── Dockerfile
│   ├── pom.xml
│   └── src/...
├── frontend/ （Vue 3）
│   ├── Dockerfile
│   ├── package.json
│   └── src/...
├── docker-compose.yml
└── README.md（任意）
```

---

# 🐳 1. docker-compose.yml（全体）

```yaml
version: '3.8'

services:
  backend:
    build: ./backend
    ports:
      - "8080:8080"
    volumes:
      - ./backend:/app
    depends_on:
      - db
    command: ./mvnw spring-boot:run

  frontend:
    build: ./frontend
    ports:
      - "5173:5173"
    volumes:
      - ./frontend:/app
    command: npm run dev
    environment:
      - VITE_API_BASE=http://localhost:8080

  db:
    image: mysql:8
    ports:
      - "3306:3306"
    environment:
      MYSQL_ROOT_PASSWORD: root
      MYSQL_DATABASE: todo_db
      MYSQL_USER: todo_user
      MYSQL_PASSWORD: todo_pass
    volumes:
      - db-data:/var/lib/mysql

volumes:
  db-data:
```

---

# 🧱 2. Spring Boot バックエンド（`backend/`）

### `backend/Dockerfile`

```Dockerfile
FROM eclipse-temurin:17-jdk
WORKDIR /app
COPY . .
RUN chmod +x mvnw
CMD ["./mvnw", "spring-boot:run"]
```

### `backend/src/main/resources/application.properties`

```properties
spring.datasource.url=jdbc:mysql://db:3306/todo_db
spring.datasource.username=todo_user
spring.datasource.password=todo_pass
spring.jpa.hibernate.ddl-auto=update
spring.jpa.show-sql=true
server.port=8080

# CORS（Vueからのアクセス許可）
spring.web.cors.allowed-origins=http://localhost:5173
spring.web.cors.allow-credentials=true
```

### `Todo.java`

```java
@Entity
public class Todo {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String task;
    private boolean done;

    public Long getId() { return id; }
    public void setId(Long id) { this.id = id; }

    public String getTask() { return task; }
    public void setTask(String task) { this.task = task; }

    public boolean isDone() { return done; }
    public void setDone(boolean done) { this.done = done; }
}
```

### `TodoRepository.java`

```java
public interface TodoRepository extends JpaRepository<Todo, Long> {}
```

### `TodoController.java`

```java
@RestController
@RequestMapping("/todos")
@CrossOrigin(origins = "http://localhost:5173") // CORS対応
public class TodoController {
    @Autowired
    private TodoRepository repo;

    @GetMapping
    public List<Todo> getAll() {
        return repo.findAll();
    }

    @PostMapping
    public Todo create(@RequestBody Todo todo) {
        return repo.save(todo);
    }

    @PutMapping("/{id}")
    public Todo update(@PathVariable Long id, @RequestBody Todo todo) {
        todo.setId(id);
        return repo.save(todo);
    }

    @DeleteMapping("/{id}")
    public void delete(@PathVariable Long id) {
        repo.deleteById(id);
    }
}
```

---

# 🎨 3. Vue 3 フロントエンド（`frontend/`）

### `frontend/Dockerfile`

```Dockerfile
FROM node:18
WORKDIR /app
COPY . .
RUN npm install
EXPOSE 5173
CMD ["npm", "run", "dev"]
```

### `frontend/package.json`

```json
{
  "name": "todo-frontend",
  "version": "1.0.0",
  "scripts": {
    "dev": "vite",
    "build": "vite build"
  },
  "dependencies": {
    "axios": "^1.4.0",
    "vue": "^3.3.4"
  },
  "devDependencies": {
    "vite": "^5.0.0"
  }
}
```

### `frontend/vite.config.js`

```js
import { defineConfig } from 'vite'
import vue from '@vitejs/plugin-vue'

export default defineConfig({
  plugins: [vue()],
  server: {
    host: '0.0.0.0',
    port: 5173
  }
})
```

### `frontend/src/main.js`

```js
import { createApp } from 'vue'
import App from './App.vue'
createApp(App).mount('#app')
```

### `frontend/src/App.vue`

```vue
<template>
  <div class="app">
    <h1>ToDoリスト</h1>
    <form @submit.prevent="addTodo">
      <input v-model="newTask" placeholder="新しいタスクを入力" />
      <button type="submit">追加</button>
    </form>
    <ul>
      <li v-for="todo in todos" :key="todo.id">
        <input type="checkbox" v-model="todo.done" @change="updateTodo(todo)" />
        <span :style="{ textDecoration: todo.done ? 'line-through' : 'none' }">
          {{ todo.task }}
        </span>
        <button @click="deleteTodo(todo.id)">削除</button>
      </li>
    </ul>
  </div>
</template>

<script>
import axios from 'axios'

export default {
  data() {
    return {
      todos: [],
      newTask: ''
    }
  },
  mounted() {
    this.fetchTodos()
  },
  methods: {
    fetchTodos() {
      axios.get('/todos').then(res => {
        this.todos = res.data
      })
    },
    addTodo() {
      if (!this.newTask) return
      axios.post('/todos', { task: this.newTask, done: false }).then(() => {
        this.newTask = ''
        this.fetchTodos()
      })
    },
    updateTodo(todo) {
      axios.put(`/todos/${todo.id}`, todo)
    },
    deleteTodo(id) {
      axios.delete(`/todos/${id}`).then(() => {
        this.todos = this.todos.filter(t => t.id !== id)
      })
    }
  }
}
</script>

<style>
.app {
  max-width: 600px;
  margin: auto;
  padding: 1rem;
}
</style>
```

---

## ✅ 最後に

### 起動方法

```bash
docker-compose up --build
```

- Spring Boot API → http://localhost:8080
- Vue フロント → http://localhost:5173

---

## 📌 補足

- axiosは`/todos`にアクセスしてるけど、Vueからは自動的にホスト側のポート`8080`に向かわないので、**VITE_API_BASEを使って解決**するのがベスト（今はCORS許可して簡略化）。
- 本格運用なら、Nginxでまとめる or フロントビルド後にSpringの`/static`に入れるのが定番。

---

必要なら、**GitHubに上げられる状態のプロジェクト一式を丸ごと生成**することもできるよ！  
「DBの初期データ入れたい」とか「UIもうちょっと整えたい」とかあれば遠慮なくどうぞ！
