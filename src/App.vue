<template>
  <div id="app">
    <div v-if="authState !== 'signedin'">
      <h1>サインイン画面</h1>
      <amplify-authenticator>
        <!-- eslint-disable-next-line vue/no-deprecated-slot-attribute -->
        <amplify-sign-up slot="sign-up"
          :form-fields="formFields"
        ></amplify-sign-up>        
      </amplify-authenticator>
    </div>
    <div v-if="authState === 'signedin' && user">
      <div>
        <h1>Todo App</h1>
        <input type="text" v-model="name" placeholder="Todo name">
        <input type="text" v-model="description" placeholder="Todo description">
        <button v-on:click="createTodo">Create Todo</button>
        <table align="center">
          <thead>
            <tr>
              <th>name</th>
              <th>description</th>
              <th>get</th>
              <th>update</th>
              <th>delete</th>
            </tr>
          </thead>
          <tbody>
            <template v-for="item in todos" :key="item.id">
              <tr>
                <td>{{ item.name }}</td>
                <td>{{ item.description }}</td>
                <td>
                  <button @click="getTodo(item.id)">get</button>
                </td>
                <td>
                  <button @click="updateTodo(item.id)">update</button>
                </td>
                <td>
                  <button @click="deleteTodo(item.id)">Delete</button>
                </td>
              </tr>
            </template>
          </tbody>
        </table>
      </div>
      <amplify-sign-out></amplify-sign-out>
    </div>
  </div>
</template>

<script>
import { API } from 'aws-amplify';
import { createTodo, deleteTodo, updateTodo } from './graphql/mutations';
import { listTodos, getTodo } from './graphql/queries';
import { onCreateTodo } from './graphql/subscriptions';
import { onAuthUIStateChange } from '@aws-amplify/ui-components';

export default {
  name: 'app',
  async created(){
    this.listTodos();
    this.subscribe();
    this.unsubscribeAuth = onAuthUIStateChange((authState, authDate) => {
      this.authState = authState;
      this.user = authDate;
    })
  },
  data() {
    return {
      name: '',
      description: '',
      todos: [],
      user: undefined,
      authState: undefined,
      unsubscribeAuth: undefined,
      formFields: [
        { type: "username" },
        { type: "password" },
        { type: "email" }
      ]
    }
  },
  beforeUnmount(){
    this.unsubscribeAuth();
  },
  methods: {
    async createTodo() {
      const { name, description } = this;
      if (!name || !description) return;
      const todo = { name, description };
      this.todos = [...this.todos, todo];
      await API.graphql({
        query: createTodo,
        variables: {input: todo},
      });
      this.name = '';
      this.description = '';
    },
    async listTodos(){
      const todos = await API.graphql({
        query: listTodos
      });
      this.todos = todos.data.listTodos.items;
    },
    async getTodo(todo_id){
      const todo = await API.graphql({
        query: getTodo,
        variables: {id: todo_id},
      });
      console.log(todo);
      const message = "Todoの詳細"
        + "\nname : " + todo.data.getTodo.name 
        + "\ndescription : " + todo.data.getTodo.description
        + "\ncreatedAt : " + todo.data.getTodo.createdAt
        + "\nupdatedAt : " + todo.data.getTodo.updatedAt;
      alert(message);
    },
    async updateTodo(todo_id){
      const todo = {
        id: todo_id,
      };
      console.log(todo);
      await API.graphql({
        query: updateTodo,
        variables: {input: todo}
      });
      const message = "update todo : " + todo_id;
      alert(message);
    },
    async deleteTodo(todo_id){
      const todo = {
        id: todo_id,
      };
      await API.graphql({
        query: deleteTodo,
        variables: {input: todo},
      });
      const message = "delete todo : " + todo_id;
      alert(message);
      this.listTodos();
    },
    subscribe(){
      API.graphql({ query: onCreateTodo })
        .subscribe({
          next: (eventData) => {
            let todo = eventData.value.data.onCreateTodo;
            if (this.todos.some(item => item.name === todo.name)) return;
            this.todos = [...this.todos, todo];
          }
        });
    },
  }
};
</script>

<style>
#app {
  font-family: Avenir, Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  text-align: center;
  color: #2c3e50;
  margin-top: 60px;
}
</style>
