<template>
	<section class="todo">
		<div class="input-group mb-3">
			<div class="input-group-prepend">
				<span class="input-group-text" id="basic-addon1">待辦事項</span>
			</div>
			<input
				type="text"
				class="form-control"
				placeholder="準備要做的任務"
				v-model="newTodo"
				@keyup.enter="addTodo"
			/>
			<div class="input-group-append">
				<button class="btn btn-primary" type="button" @click="addTodo">
					新增
				</button>
			</div>
		</div>
		<div class="card text-center">
			<div class="card-header">
				<ul class="nav nav-tabs card-header-tabs">
					<li class="nav-item">
						<a
							class="nav-link"
							:class="{ active: visibility == 'all' }"
							@click="visibility = 'all'"
							href="#"
							>待辦事項</a
						>
					</li>
				</ul>
			</div>
			<ul class="list-group list-group-flush text-left">
				<li class="list-group-item" v-for="item in todos" v-bind:key="item.id">
					<div class="d-flex" v-if="item.id !== editTodo.id">
						<div class="flex-grow-1 d-flex align-items-center">
							<p :for="item.id">{{ item.task }}</p>
						</div>
						<div class="d-flex align-items-center">
							<p class="mb-0 me-1">{{ item.assignee }}</p>
							<svg
								xmlns="http://www.w3.org/2000/svg"
								width="16"
								height="16"
								fill="currentColor"
								class="bi bi-pencil-square me-2 ms-2"
								viewBox="0 0 16 16"
								@click="edit(item)"
							>
								<path
									d="M15.502 1.94a.5.5 0 0 1 0 .706L14.459 3.69l-2-2L13.502.646a.5.5 0 0 1 .707 0l1.293 1.293zm-1.75 2.456-2-2L4.939 9.21a.5.5 0 0 0-.121.196l-.805 2.414a.25.25 0 0 0 .316.316l2.414-.805a.5.5 0 0 0 .196-.12l6.813-6.814z"
								/>
								<path
									fill-rule="evenodd"
									d="M1 13.5A1.5 1.5 0 0 0 2.5 15h11a1.5 1.5 0 0 0 1.5-1.5v-6a.5.5 0 0 0-1 0v6a.5.5 0 0 1-.5.5h-11a.5.5 0 0 1-.5-.5v-11a.5.5 0 0 1 .5-.5H9a.5.5 0 0 0 0-1H2.5A1.5 1.5 0 0 0 1 2.5v11z"
								/>
							</svg>
							<button
								type="button"
								class="btn btn-success ms-2"
								aria-label="Close"
								@click="removeTodo(item)"
							>
								完成
							</button>
						</div>
					</div>
					<input
						type="text"
						class="form-control"
						v-if="item.id == editTodo.id"
						@keyup.enter="doneEdit(item)"
						v-model="editTodo.task"
					/>
				</li>
			</ul>
		</div>
	</section>
</template>

<script>
	import gql from "graphql-tag";
	import moment from "moment";

	const GET_TODOS = gql`
		query getMyTodos {
			todo_list(order_by: { created_at: desc }) {
				id
				updated_at
				task
				assignee
			}
		}
	`;
	const ADD_TODO = gql`
		mutation ($todo: String!) {
			insert_todo_list(objects: { task: $todo, assignee: "yeh" }) {
				affected_rows
				returning {
					id
					task
					updated_at
					assignee
				}
			}
		}
	`;

	const REMOVE_TODO = gql`
		mutation removeTodo($id: Int!) {
			delete_todo_list(where: { id: { _eq: $id } }) {
				affected_rows
			}
		}
	`;
	const UPDATE_TODO = gql`
		mutation updateTodo($id: Int!, $task: String) {
			update_todo_list(where: { id: { _eq: $id } }, _set: { task: $task }) {
				affected_rows
			}
		}
	`;

	export default {
		name: "TodoList",
		data() {
			return {
				newTodo: "",
				todos: [],
				editTodo: {},
				editTask: "",
				visibility: "all",
			};
		},
		apollo: {
			$subscribe: {
				tags: {
					query: gql`
						subscription {
							todo_list(
								order_by: { updated_at: desc }
								where: { assignee: { _eq: "yeh" } }
							) {
								id
								task
								updated_at
								assignee
							}
						}
					`,
					result({ data }) {
						let todoList = data.todo_list;
						todoList.forEach((todo) => {
							todo.updated_at = moment(todo.updated_at).format("M/D");
						});

						this.todos = data.todo_list;
					},
				},
			},
		},
		methods: {
			addTodo() {
				let value = this.newTodo.trim();
				//id
				let timesTamp = moment(Date.now()).format("Y-M-D h:mm a");
				//如果空白，則不能新增
				if (!value) {
					return;
				}
				// 新增物件進todos
				this.todos.unshift({
					id: timesTamp,
					task: value,
					updated_at: timesTamp,
				});
				this.$apollo.mutate({
					mutation: ADD_TODO,
					variables: {
						todo: value,
					},
				});
				this.newTodo = "";
			},

			removeTodo(item) {
				let newIndex = this.todos.findIndex(function (task) {
					return item.id === task.id;
				});

				this.todos.splice(newIndex, 1);

				this.$apollo.mutate({
					mutation: REMOVE_TODO,
					variables: {
						id: item.id,
					},
					update: (store, { data: { delete_todo_list } }) => {
						if (delete_todo_list.affected_rows) {
							if (this.type === "private") {
								const data = store.readQuery({
									query: GET_TODOS,
								});
								data.todos = data.todos.filter((t) => {
									return t.id !== item.id;
								});
								store.writeQuery({
									query: GET_TODOS,
									data,
								});
							}
						}
					},
				});
			},

			edit(item) {
				this.editTodo = item;
				this.editTask = item.task;
			},

			doneEdit(item) {
				item.task = this.editTodo.task;
				this.$apollo.mutate({
					mutation: UPDATE_TODO,
					variables: {
						id: item.id,
						task: item.task,
					},
				});
				this.editTask = "";
				this.editTodo = {};
			},
		},
	};
</script>

<style scoped>
	.todo {
		max-width: 600px;
		margin: 0 auto;
	}
	.completed {
		text-decoration: line-through;
	}

	.bi-pencil-square {
		cursor: pointer;
	}
	p {
		margin-bottom: 0;
	}
</style>
