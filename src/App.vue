<template>
	<div id="app" :class="['kanban-board', { dark: isDarkMode }]">
		<!-- Header -->
		<header class="main-header">
			<div class="logo">
				<i class="fas fa-ticket-alt"></i>
				<h1>Tickity</h1>
			</div>
			<div class="header-actions">
				<input
					type="text"
					v-model="searchQuery"
					placeholder="Search tasks..."
					class="search-bar" />
				<select v-model="assigneeFilter" class="assignee-filter">
					<option value="">All Assignees</option>
					<option
						v-for="user in users"
						:key="user.id"
						:value="user.name">
						{{ user.name }}
					</option>
				</select>
				<button @click="addColumn">+ Add Column</button>
				<button @click="toggleDarkMode">
					<i :class="isDarkMode ? 'fas fa-sun' : 'fas fa-moon'"></i>
				</button>
				<div class="profile-menu">
					<div
						class="notification-badge"
						v-if="assignedTasksCount > 0">
						{{ assignedTasksCount }}
					</div>
					<img
						:src="currentUser.avatar || defaultAvatarUrl"
						alt="Profile Avatar"
						@click="toggleProfileMenu" />
					<div v-if="showProfileMenu" class="profile-dropdown">
						<a href="#" @click.prevent="openProfileSettings"
							>Settings</a
						>
						<a href="#" @click.prevent="logout">Logout</a>
					</div>
				</div>
			</div>
		</header>

		<!-- Columns -->
		<div class="columns">
			<draggable
				v-model="columns"
				group="columns"
				class="columns-list"
				item-key="id"
				direction="horizontal"
				@end="onColumnDragEnd">
				<template #item="{ element: column, index: columnIndex }">
					<div class="column">
						<div class="column-header">
							<h2 @click="editColumn(columnIndex)">
								{{ column.name }}
							</h2>
							<button
								class="delete-column-btn"
								@click="deleteColumn(columnIndex)">
								×
							</button>
						</div>
						<draggable
							v-model="column.tasks"
							:group="'tasks'"
							class="task-list"
							item-key="id"
							@end="onDragEnd">
							<template
								#item="{ element: task, index: taskIndex }">
								<transition
									name="task-transition"
									mode="out-in">
									<div
										v-if="matchesSearch(task)"
										class="task"
										:class="{
											overdue: isOverdue(task),
											'due-soon': isDueSoon(task),
											'assigned-to-me':
												task.assignee ===
												currentUser.name,
										}"
										@click="editTask(column, taskIndex)">
										<div class="task-header">
											<div class="avatars">
												<div class="owner-avatar">
													<img
														:src="
															getUserAvatar(
																task.owner
															) ||
															defaultAvatarUrl
														"
														alt="Owner Avatar" />
												</div>
												<div
													class="assignee-avatar"
													v-if="
														task.assignee &&
														task.assignee !==
															task.owner
													">
													<img
														:src="
															getUserAvatar(
																task.assignee
															)
														"
														alt="Assignee Avatar" />
												</div>
											</div>
											<div class="task-info">
												<h3>{{ task.title }}</h3>
												<div class="task-tags">
													<span
														v-for="tag in task.tags"
														:key="tag.id"
														class="tag"
														:style="{
															backgroundColor:
																tag.color,
														}">
														{{ tag.name }}
													</span>
													<span
														v-if="task.epic"
														class="epic-tag"
														>{{ task.epic }}</span
													>
													<span class="priority-icon">
														<i
															:class="{
																'fas fa-arrow-down low-priority':
																	task.priority ===
																	'Low',
																'fas fa-minus medium-priority':
																	task.priority ===
																	'Medium',
																'fas fa-arrow-up high-priority':
																	task.priority ===
																	'High',
															}"></i>
													</span>
												</div>
											</div>
											<button
												class="delete-task-btn"
												@click.stop="
													deleteTask(
														column,
														taskIndex
													)
												">
												×
											</button>
										</div>
										<div class="task-footer">
											<p
												class="due-date"
												v-if="task.dueDate">
												<i
													class="far fa-calendar-alt"></i>
												{{ task.dueDate }}
											</p>
											<div class="progress-bar">
												<div
													class="progress"
													:style="{
														width:
															task.progress + '%',
													}"></div>
											</div>
										</div>
									</div>
								</transition>
							</template>
						</draggable>
						<button class="add-task-btn" @click="addTask(column)">
							+ Add Task
						</button>
					</div>
				</template>
			</draggable>
		</div>

		<!-- Edit Modal -->
		<div
			v-if="showModal && modalItem"
			class="modal-overlay"
			@click.self="closeModal">
			<div class="modal">
				<div class="modal-header">
					<input
						v-if="isEditingColumn"
						v-model="modalItem.name"
						class="modal-title-input"
						placeholder="Column Name" />
					<input
						v-else
						v-model="modalItem.title"
						class="modal-title-input"
						placeholder="Task Title" />
					<button class="close-modal-btn" @click="closeModal">
						×
					</button>
				</div>
				<div class="modal-body">
					<div class="modal-content">
						<!-- Left Section -->
						<div class="modal-left" v-if="!isEditingColumn">
							<!-- Description -->
							<div class="modal-section">
								<h3>Description</h3>
								<textarea
									v-model="modalItem.description"
									class="modal-description"></textarea>
							</div>
							<!-- Subtasks -->
							<div class="modal-section">
								<h3>Subtasks</h3>
								<ul class="subtasks-list">
									<li
										v-for="(
											subtask, index
										) in modalItem.subtasks"
										:key="index">
										<input
											type="checkbox"
											v-model="subtask.completed" />
										<input
											v-model="subtask.title"
											class="subtask-input"
											placeholder="Subtask title" />
										<button
											class="delete-subtask-btn"
											@click="deleteSubtask(index)">
											×
										</button>
									</li>
								</ul>
								<button
									class="add-subtask-btn"
									@click="addSubtask">
									+ Add Subtask
								</button>
							</div>
							<!-- Attachments -->
							<div class="modal-section">
								<h3>Attachments</h3>
								<div class="attachments-list">
									<div
										v-for="(
											file, index
										) in modalItem.attachments"
										:key="index"
										class="attachment">
										<div
											v-if="
												file.type.startsWith('image/')
											">
											<img
												:src="file.url"
												alt="Attachment Image"
												class="attachment-image" />
										</div>
										<div v-else>
											<i class="fas fa-paperclip"></i>
											{{ file.name }}
										</div>
									</div>
								</div>
								<input type="file" @change="handleFileUpload" />
							</div>
						</div>
						<!-- Right Section -->
						<div class="modal-right" v-if="!isEditingColumn">
							<!-- Details -->
							<div class="modal-section">
								<h3>Details</h3>
								<div class="modal-field">
									<label>Epic</label>
									<input
										v-model="modalItem.epic"
										placeholder="Epic" />
								</div>
								<div class="modal-field">
									<label>Due Date</label>
									<input
										v-model="modalItem.dueDate"
										type="date"
										placeholder="Due Date" />
								</div>
								<div class="modal-field">
									<label>Priority</label>
									<select v-model="modalItem.priority">
										<option disabled value="">
											Priority
										</option>
										<option>Low</option>
										<option>Medium</option>
										<option>High</option>
									</select>
								</div>
								<div class="modal-field">
									<label>Tags</label>
									<div class="tags-input">
										<div
											v-for="(
												tag, index
											) in modalItem.tags"
											:key="tag.id"
											class="tag"
											:style="{
												backgroundColor: tag.color,
											}">
											{{ tag.name }}
											<span
												class="remove-tag"
												@click="removeTag(index)"
												>×</span
											>
										</div>
										<input
											v-model="newTagName"
											placeholder="Add a tag..."
											@keyup.enter="addTag" />
										<input
											v-model="newTagColor"
											type="color" />
									</div>
								</div>
								<div class="modal-field">
									<label>Assign To</label>
									<select v-model="modalItem.assignee">
										<option
											v-for="user in users"
											:key="user.id"
											:value="user.name">
											{{ user.name }}
										</option>
									</select>
								</div>
								<div class="modal-field">
									<label>Owner</label>
									<select v-model="modalItem.owner">
										<option
											v-for="user in users"
											:key="user.id"
											:value="user.name">
											{{ user.name }}
										</option>
									</select>
								</div>
								<div class="modal-field">
									<label>Progress (%)</label>
									<input
										v-model.number="modalItem.progress"
										type="number"
										min="0"
										max="100" />
								</div>
							</div>
						</div>
					</div>
					<!-- Chat Section -->
					<div
						class="modal-section chat-section"
						v-if="!isEditingColumn">
						<h3>Comments</h3>
						<div class="chat-container">
							<div
								v-for="comment in modalItem.comments"
								:key="comment.id"
								:class="[
									'chat-message',
									{
										'own-message':
											comment.author === currentUser.name,
									},
								]">
								<div class="chat-avatar">
									<img
										:src="
											getUserAvatar(comment.author) ||
											defaultAvatarUrl
										"
										alt="Author Avatar" />
								</div>
								<div class="chat-bubble">
									<div class="chat-author">
										{{ comment.author }}
									</div>
									<div class="chat-text">
										{{ comment.text }}
									</div>
									<div class="chat-time">
										{{ formatDate(comment.date) }}
									</div>
								</div>
							</div>
						</div>
						<div class="chat-input">
							<textarea
								v-model="newCommentText"
								placeholder="Type a message..."></textarea>
							<button @click="addComment">
								<i class="fas fa-paper-plane"></i>
							</button>
						</div>
					</div>
					<!-- Activity Log -->
					<div class="modal-section" v-if="!isEditingColumn">
						<h3>Activity Log</h3>
						<ul class="activity-log">
							<li
								v-for="(
									activity, index
								) in modalItem.activityLog"
								:key="index">
								{{ formatDate(activity.date) }} -
								{{ activity.description }}
							</li>
						</ul>
					</div>
					<!-- Modal Actions -->
					<div class="modal-actions">
						<button @click="saveItem">Save Changes</button>
						<button @click="closeModal">Cancel</button>
					</div>
				</div>
			</div>
		</div>

		<!-- Profile Settings Modal -->
		<div
			v-if="showProfileSettings"
			class="modal-overlay"
			@click.self="closeProfileSettings">
			<div class="modal">
				<div class="modal-header">
					<h2>Profile Settings</h2>
					<button
						class="close-modal-btn"
						@click="closeProfileSettings">
						×
					</button>
				</div>
				<div class="modal-body">
					<div class="modal-field">
						<label>Display Name</label>
						<input v-model="currentUser.name" />
					</div>
					<div class="modal-field">
						<label>Email</label>
						<input v-model="currentUser.email" type="email" />
					</div>
					<div class="modal-field">
						<label>Avatar</label>
						<input type="file" @change="updateAvatar" />
						<div class="avatar-preview">
							<img
								:src="currentUser.avatar || defaultAvatarUrl"
								alt="Avatar Preview" />
						</div>
					</div>
					<div class="modal-field">
						<label>Theme</label>
						<select v-model="currentUser.theme">
							<option value="light">Light</option>
							<option value="dark">Dark</option>
							<option value="system">System Default</option>
						</select>
					</div>
					<div class="modal-actions">
						<button @click="saveProfileSettings">Save</button>
						<button @click="closeProfileSettings">Cancel</button>
					</div>
				</div>
			</div>
		</div>

		<!-- Login Screen -->
		<div v-if="!isLoggedIn" class="login-screen">
			<div class="login-container">
				<h2>Welcome to Tickity</h2>
				<input v-model="loginName" placeholder="Enter your name" />
				<button @click="login">Login</button>
			</div>
		</div>

		<!-- Footer -->
		<footer class="main-footer">
			<p>&copy; 2024 Tickity. All rights reserved.</p>
			<div class="footer-links">
				<a href="#">Privacy Policy</a>
				<a href="#">Terms of Service</a>
				<a href="#">Contact Us</a>
			</div>
		</footer>
	</div>
</template>

<script>
import { defineComponent } from "vue";
import draggable from "vuedraggable";

export default defineComponent({
	name: "App",
	components: {
		draggable,
	},
	data() {
		return {
			isDarkMode: false,
			showModal: false,
			isEditingColumn: false,
			showProfileMenu: false,
			showProfileSettings: false,
			isLoggedIn: false,
			loginName: "",
			currentUser: {
				name: "",
				avatar: "",
				email: "",
				theme: "light",
			},
			modalItem: null,
			modalColumn: null,
			modalItemIndex: null,
			defaultAvatarUrl: "https://randomuser.me/api/portraits/lego/1.jpg",
			newCommentText: "",
			newTagName: "",
			newTagColor: "#17a2b8",
			searchQuery: "",
			assigneeFilter: "",
			users: [
				{
					id: 1,
					name: "Alice",
					avatar: "https://randomuser.me/api/portraits/women/68.jpg",
				},
				{
					id: 2,
					name: "Bob",
					avatar: "https://randomuser.me/api/portraits/men/65.jpg",
				},
				{
					id: 3,
					name: "Charlie",
					avatar: "https://randomuser.me/api/portraits/men/66.jpg",
				},
			],
			columns: [
				{
					id: 1,
					name: "To Do",
					tasks: [
						{
							id: 1,
							title: "Implement Authentication",
							epic: "User Management",
							dueDate: "2024-11-20",
							priority: "High",
							owner: "Alice",
							description:
								"Set up user authentication using JWT tokens.",
							comments: [],
							assignee: "Bob",
							attachments: [],
							progress: 0,
							tags: [
								{ id: 1, name: "Backend", color: "#007bff" },
								{ id: 2, name: "Auth", color: "#17a2b8" },
							],
							activityLog: [],
							subtasks: [
								{
									title: "Design database schema",
									completed: false,
								},
								{
									title: "Implement login endpoint",
									completed: false,
								},
								{
									title: "Set up JWT middleware",
									completed: false,
								},
							],
						},
						{
							id: 2,
							title: "Set Up CI/CD Pipeline",
							epic: "Deployment",
							dueDate: "2024-11-25",
							priority: "Medium",
							owner: "Charlie",
							description:
								"Configure CI/CD pipeline for automated testing and deployment.",
							comments: [],
							assignee: "Alice",
							attachments: [],
							progress: 0,
							tags: [{ id: 3, name: "DevOps", color: "#6f42c1" }],
							activityLog: [],
							subtasks: [
								{
									title: "Set up GitHub Actions",
									completed: false,
								},
								{
									title: "Write deployment scripts",
									completed: false,
								},
							],
						},
					],
				},
				{
					id: 2,
					name: "In Progress",
					tasks: [
						{
							id: 3,
							title: "Create REST API",
							epic: "Backend Development",
							dueDate: "2024-11-18",
							priority: "High",
							owner: "Bob",
							description:
								"Develop RESTful API endpoints for the application.",
							comments: [],
							assignee: "Bob",
							attachments: [],
							progress: 50,
							tags: [{ id: 4, name: "API", color: "#28a745" }],
							activityLog: [],
							subtasks: [
								{
									title: "Implement GET endpoints",
									completed: true,
								},
								{
									title: "Implement POST endpoints",
									completed: false,
								},
								{
									title: "Implement PUT/PATCH endpoints",
									completed: false,
								},
								{
									title: "Implement DELETE endpoints",
									completed: false,
								},
							],
						},
					],
				},
				{
					id: 3,
					name: "Done",
					tasks: [
						{
							id: 4,
							title: "Design Database Schema",
							epic: "Database",
							dueDate: "2024-11-10",
							priority: "Low",
							owner: "Alice",
							description:
								"Design the initial database schema for the project.",
							comments: [],
							assignee: "Charlie",
							attachments: [],
							progress: 100,
							tags: [
								{ id: 5, name: "Database", color: "#fd7e14" },
							],
							activityLog: [],
							subtasks: [
								{ title: "Identify entities", completed: true },
								{
									title: "Define relationships",
									completed: true,
								},
								{ title: "Create ER diagram", completed: true },
							],
						},
					],
				},
				{
					id: 4,
					name: "Blocked",
					tasks: [
						{
							id: 5,
							title: "Integrate Payment Gateway",
							epic: "Payment",
							dueDate: "2024-11-30",
							priority: "High",
							owner: "Charlie",
							description:
								"Integrate Stripe payment gateway for processing payments.",
							comments: [
								{
									id: 1,
									author: "Charlie",
									text: "Waiting on API keys from finance team.",
									date: "2024-11-12T09:00:00",
								},
							],
							assignee: "Alice",
							attachments: [],
							progress: 0,
							tags: [
								{
									id: 6,
									name: "Integration",
									color: "#dc3545",
								},
							],
							activityLog: [],
							subtasks: [
								{
									title: "Set up test environment",
									completed: false,
								},
								{
									title: "Implement payment processing",
									completed: false,
								},
							],
						},
					],
				},
			],
		};
	},
	computed: {
		assignedTasksCount() {
			return this.columns.reduce((count, column) => {
				return (
					count +
					column.tasks.filter(
						task => task.assignee === this.currentUser.name
					).length
				);
			}, 0);
		},
	},
	methods: {
		toggleDarkMode() {
			this.isDarkMode = !this.isDarkMode;
			this.currentUser.theme = this.isDarkMode ? "dark" : "light";
			this.saveUserSettings();
		},
		toggleProfileMenu() {
			this.showProfileMenu = !this.showProfileMenu;
		},
		openProfileSettings() {
			this.showProfileSettings = true;
			this.showProfileMenu = false;
		},
		closeProfileSettings() {
			this.showProfileSettings = false;
		},
		saveProfileSettings() {
			this.isDarkMode = this.currentUser.theme === "dark";
			this.saveUserSettings();
			this.closeProfileSettings();
		},
		updateAvatar(event) {
			const file = event.target.files[0];
			if (file) {
				const reader = new FileReader();
				reader.onload = e => {
					this.currentUser.avatar = e.target.result;
					this.saveUserSettings();
				};
				reader.readAsDataURL(file);
			}
		},
		saveUserSettings() {
			localStorage.setItem(
				"currentUser",
				JSON.stringify(this.currentUser)
			);
		},
		loadUserSettings() {
			const data = localStorage.getItem("currentUser");
			if (data) {
				this.currentUser = JSON.parse(data);
				this.isDarkMode = this.currentUser.theme === "dark";
			}
		},
		logout() {
			this.isLoggedIn = false;
			this.currentUser = {
				name: "",
				avatar: "",
				email: "",
				theme: "light",
			};
			localStorage.removeItem("currentUser");
		},
		login() {
			if (this.loginName.trim() !== "") {
				this.currentUser.name = this.loginName.trim();
				this.isLoggedIn = true;
				this.users.push({
					id: Date.now(),
					name: this.currentUser.name,
					avatar: this.currentUser.avatar || this.defaultAvatarUrl,
				});
				this.saveUserSettings();
			}
		},
		onDragEnd() {
			this.saveToLocalStorage();
		},
		onColumnDragEnd() {
			this.saveToLocalStorage();
		},
		addTask(column) {
			const newTask = {
				id: Date.now(),
				title: "New Task",
				epic: "",
				dueDate: "",
				priority: "",
				owner: this.currentUser.name,
				description: "",
				comments: [],
				assignee: "",
				attachments: [],
				progress: 0,
				tags: [],
				activityLog: [],
				subtasks: [],
			};
			column.tasks.push(newTask);
			this.saveToLocalStorage();
		},
		editTask(column, index) {
			this.isEditingColumn = false;
			this.modalItem = column.tasks[index];
			this.modalColumn = column;
			this.modalItemIndex = index;
			// Initialize arrays if undefined
			if (!this.modalItem.comments) {
				this.modalItem.comments = [];
			}
			if (!this.modalItem.tags) {
				this.modalItem.tags = [];
			}
			if (!this.modalItem.activityLog) {
				this.modalItem.activityLog = [];
			}
			if (!this.modalItem.subtasks) {
				this.modalItem.subtasks = [];
			}
			this.showModal = true;
			this.$nextTick(() => {
				const chatContainer = this.$el.querySelector(".chat-container");
				if (chatContainer) {
					chatContainer.scrollTop = chatContainer.scrollHeight;
				}
			});
		},
		deleteTask(column, index) {
			column.tasks.splice(index, 1);
			this.saveToLocalStorage();
		},
		addColumn() {
			const newColumn = {
				id: Date.now(),
				name: "New Column",
				tasks: [],
			};
			this.columns.push(newColumn);
			this.saveToLocalStorage();
		},
		editColumn(index) {
			this.isEditingColumn = true;
			this.modalItem = this.columns[index];
			this.modalItemIndex = index;
			// Set modalItem.name to modalItem.title for consistent binding
			this.modalItem.title = this.modalItem.name;
			this.showModal = true;
		},
		deleteColumn(index) {
			this.columns.splice(index, 1);
			this.saveToLocalStorage();
		},
		saveItem() {
			if (this.isEditingColumn) {
				// Update the column name
				this.columns[this.modalItemIndex].name = this.modalItem.name;
			} else {
				// Add to activity log
				this.modalItem.activityLog.push({
					date: new Date().toISOString(),
					description: `${this.currentUser.name} updated the task details.`,
				});
			}
			this.saveToLocalStorage();
			this.closeModal();
		},
		closeModal() {
			this.showModal = false;
			this.modalItem = null;
			this.newCommentText = "";
			this.newTagName = "";
			this.newTagColor = "#17a2b8";
		},
		saveToLocalStorage() {
			localStorage.setItem("kanbanData", JSON.stringify(this.columns));
		},
		loadFromLocalStorage() {
			const data = localStorage.getItem("kanbanData");
			if (data) {
				this.columns = JSON.parse(data);
			}
		},
		addComment() {
			if (this.newCommentText.trim() !== "") {
				if (!this.modalItem.comments) {
					this.modalItem.comments = [];
				}
				const newComment = {
					id: Date.now(),
					author: this.currentUser.name,
					text: this.newCommentText,
					date: new Date().toISOString(),
				};
				this.modalItem.comments.push(newComment);
				// Add to activity log
				this.modalItem.activityLog.push({
					date: new Date().toISOString(),
					description: `${this.currentUser.name} added a new comment.`,
				});
				this.newCommentText = "";
				this.$nextTick(() => {
					const chatContainer =
						this.$el.querySelector(".chat-container");
					if (chatContainer) {
						chatContainer.scrollTop = chatContainer.scrollHeight;
					}
				});
				this.saveToLocalStorage();
			}
		},
		matchesSearch(task) {
			const matchesTitle = task.title
				.toLowerCase()
				.includes(this.searchQuery.toLowerCase());
			const matchesAssignee = this.assigneeFilter
				? task.assignee === this.assigneeFilter
				: true;
			return matchesTitle && matchesAssignee;
		},
		isOverdue(task) {
			const today = new Date().toISOString().split("T")[0];
			return task.dueDate && task.dueDate < today;
		},
		isDueSoon(task) {
			const today = new Date();
			const dueDate = new Date(task.dueDate);
			const diffTime = dueDate - today;
			const diffDays = Math.ceil(diffTime / (1000 * 60 * 60 * 24));
			return diffDays > 0 && diffDays <= 3;
		},
		handleFileUpload(event) {
			const file = event.target.files[0];
			if (file) {
				const reader = new FileReader();
				reader.onload = e => {
					this.modalItem.attachments.push({
						name: file.name,
						url: e.target.result,
						type: file.type,
					});
					// Add to activity log
					this.modalItem.activityLog.push({
						date: new Date().toISOString(),
						description: `${this.currentUser.name} added an attachment.`,
					});
					this.saveToLocalStorage();
				};
				reader.readAsDataURL(file);
			}
		},
		getUserAvatar(userName) {
			const user = this.users.find(user => user.name === userName);
			return user ? user.avatar : this.defaultAvatarUrl;
		},
		formatDate(dateString) {
			const options = {
				year: "numeric",
				month: "short",
				day: "numeric",
				hour: "numeric",
				minute: "numeric",
			};
			return new Date(dateString).toLocaleString(undefined, options);
		},
		addTag() {
			if (this.newTagName.trim() !== "") {
				if (!this.modalItem.tags) {
					this.modalItem.tags = [];
				}
				this.modalItem.tags.push({
					id: Date.now(),
					name: this.newTagName,
					color: this.newTagColor,
				});
				this.newTagName = "";
				this.newTagColor = "#17a2b8";
				// Add to activity log
				this.modalItem.activityLog.push({
					date: new Date().toISOString(),
					description: `${this.currentUser.name} added a new tag.`,
				});
				this.saveToLocalStorage();
			}
		},
		removeTag(index) {
			this.modalItem.tags.splice(index, 1);
			// Add to activity log
			this.modalItem.activityLog.push({
				date: new Date().toISOString(),
				description: `${this.currentUser.name} removed a tag.`,
			});
			this.saveToLocalStorage();
		},
		addSubtask() {
			this.modalItem.subtasks.push({ title: "", completed: false });
		},
		deleteSubtask(index) {
			this.modalItem.subtasks.splice(index, 1);
		},
	},
	mounted() {
		this.loadFromLocalStorage();
		this.loadUserSettings();
		if (this.currentUser.name) {
			this.isLoggedIn = true;
		}
	},
});
</script>

<style scoped>
/* General Styles */
@import url("https://fonts.googleapis.com/css2?family=Roboto:wght@400;500;700&display=swap");
@import url("https://cdnjs.cloudflare.com/ajax/libs/font-awesome/5.15.4/css/all.min.css");

* {
	box-sizing: border-box;
}

html,
body,
#app {
	margin: 0;
	padding: 0;
	height: 100%;
}

#app {
	display: flex;
	flex-direction: column;
	width: 100%;
}

/* Kanban Board */
.kanban-board {
	font-family: "Roboto", sans-serif;
	background-color: #f8f9fa;
	color: #212529;
	flex: 1;
	display: flex;
	flex-direction: column;
	transition: background-color 0.3s ease;
}

.kanban-board.dark {
	background-color: #1a1a1a;
	color: #f8f9fa;
}

/* Header */
.main-header {
	position: fixed;
	top: 0;
	width: 100%;
	z-index: 1000;
	padding: 10px 20px;
	background-color: #fff;
	border-bottom: 1px solid #dee2e6;
	display: flex;
	justify-content: space-between;
	align-items: center;
}

.kanban-board.dark .main-header {
	background-color: #2a2a2a;
	border-bottom-color: #444;
}

.logo {
	display: flex;
	align-items: center;
}

.logo i {
	font-size: 2rem;
	color: #28a745;
	margin-right: 10px;
}

.logo h1 {
	font-size: 1.8rem;
	margin: 0;
	color: #28a745;
}

.header-actions {
	display: flex;
	align-items: center;
}

.header-actions button {
	background: none;
	border: none;
	color: inherit;
	font-size: 1.5rem;
	cursor: pointer;
	margin-left: 10px;
}

.header-actions button:hover {
	color: #28a745;
}

.assignee-filter {
	padding: 10px;
	border: none;
	border-radius: 30px;
	margin-right: 10px;
	background-color: #f1f3f5;
	color: #212529;
	appearance: none;
}

.kanban-board.dark .assignee-filter {
	background-color: #3a3a3a;
	color: #f8f9fa;
}

.assignee-filter:focus {
	outline: none;
	box-shadow: 0 0 0 2px rgba(40, 167, 69, 0.25);
}

.kanban-board.dark .assignee-filter:focus {
	box-shadow: 0 0 0 2px rgba(40, 167, 69, 0.5);
}

.search-bar {
	padding: 10px;
	border: none;
	border-radius: 30px;
	margin-right: 10px;
	width: 200px;
	background-color: #f1f3f5;
	color: #212529;
	transition: width 0.3s ease;
}

.kanban-board.dark .search-bar {
	background-color: #3a3a3a;
	color: #f8f9fa;
}

.search-bar:focus {
	width: 300px;
	outline: none;
	box-shadow: 0 0 0 2px rgba(40, 167, 69, 0.25);
}

.kanban-board.dark .search-bar:focus {
	box-shadow: 0 0 0 2px rgba(40, 167, 69, 0.5);
}

/* Profile Menu */
.profile-menu {
	position: relative;
	margin-left: 10px;
}

.profile-menu img {
	width: 40px;
	height: 40px;
	border-radius: 50%;
	cursor: pointer;
}

.notification-badge {
	position: absolute;
	top: -5px;
	right: -5px;
	background-color: #dc3545;
	color: #fff;
	border-radius: 50%;
	padding: 2px 6px;
	font-size: 0.8rem;
}

.profile-dropdown {
	position: absolute;
	right: 0;
	top: 50px;
	background-color: #fff;
	border: 1px solid #dee2e6;
	border-radius: 4px;
	overflow: hidden;
	box-shadow: 0 2px 10px rgba(0, 0, 0, 0.1);
}

.profile-dropdown a {
	display: block;
	padding: 10px 15px;
	color: #212529;
	text-decoration: none;
}

.profile-dropdown a:hover {
	background-color: #f1f3f5;
}

.kanban-board.dark .profile-dropdown {
	background-color: #2a2a2a;
	border-color: #444;
}

.kanban-board.dark .profile-dropdown a {
	color: #f8f9fa;
}

.kanban-board.dark .profile-dropdown a:hover {
	background-color: #333;
}

/* Columns and Tasks */
.columns {
	flex: 1;
	display: flex;
	padding: 80px 20px 20px; /* Adjust padding-top based on header height */
	overflow-x: auto;
}

.columns-list {
	display: flex;
	flex-direction: row;
}

.column {
	background-color: #e9ecef;
	border-radius: 8px;
	padding: 15px;
	flex: 0 0 350px;
	margin-right: 20px;
	display: flex;
	flex-direction: column;
	transition: background-color 0.3s ease, transform 0.3s ease;
}

.column:hover {
	background-color: #dee2e6;
	transform: scale(1.01);
}

.column:last-child {
	margin-right: 0;
}

.kanban-board.dark .column {
	background-color: #2a2a2a;
}

.kanban-board.dark .column:hover {
	background-color: #333;
}

.column-header {
	display: flex;
	justify-content: space-between;
	align-items: center;
}

.column-header h2 {
	font-size: 1.8rem;
	margin: 0 0 15px;
	cursor: pointer;
	transition: color 0.3s ease;
}

.column-header h2:hover {
	color: #28a745;
}

.kanban-board.dark .column-header h2:hover {
	color: #28a745;
}

.delete-column-btn {
	background: none;
	border: none;
	font-size: 1.5rem;
	cursor: pointer;
	color: inherit;
	transition: color 0.3s ease;
}

.delete-column-btn:hover {
	color: #dc3545;
}

.task-list {
	flex: 1;
	overflow-y: auto;
}

.task {
	background-color: #fff;
	border-radius: 6px;
	padding: 15px;
	margin-bottom: 15px;
	cursor: pointer;
	transition: background-color 0.3s ease, transform 0.3s ease,
		box-shadow 0.3s ease;
	position: relative;
}

.kanban-board.dark .task {
	background-color: #3a3a3a;
}

.task:hover {
	background-color: #f1f3f5;
	transform: translateY(-3px);
	box-shadow: 0 2px 10px rgba(0, 0, 0, 0.1);
}

.kanban-board.dark .task:hover {
	background-color: #444;
}

.overdue {
	border-left: 5px solid #dc3545;
}

.due-soon {
	border-left: 5px solid #ffc107;
}

.assigned-to-me::after {
	content: "Assigned to you";
	position: absolute;
	top: 10px;
	right: 10px;
	background-color: #28a745;
	color: #fff;
	padding: 2px 6px;
	border-radius: 4px;
	font-size: 0.8rem;
}

.task-header {
	display: flex;
	align-items: center;
}

.avatars {
	display: flex;
	align-items: center;
}

.owner-avatar {
	width: 30px;
	height: 30px;
	border-radius: 50%;
	overflow: hidden;
}

.assignee-avatar {
	width: 40px;
	height: 40px;
	border-radius: 50%;
	overflow: hidden;
	margin-left: -10px; /* Overlap avatars */
	border: 2px solid #fff;
}

.kanban-board.dark .assignee-avatar {
	border-color: #2a2a2a;
}

.owner-avatar img,
.assignee-avatar img {
	width: 100%;
	height: auto;
}

.task-info {
	flex: 1;
	margin-left: 10px;
}

.task-header h3 {
	margin: 0;
	font-size: 1.2rem;
	transition: color 0.3s ease;
}

.task-header h3:hover {
	color: #28a745;
}

.task-tags {
	margin-top: 5px;
}

.label-tag,
.tag {
	display: inline-block;
	padding: 2px 6px;
	border-radius: 4px;
	font-size: 0.8rem;
	margin-right: 5px;
	color: #fff;
}

.epic-tag {
	display: inline-block;
	padding: 2px 6px;
	border-radius: 4px;
	font-size: 0.8rem;
	margin-right: 5px;
	color: #fff;
	background-color: #6f42c1;
}

.priority-icon {
	margin-right: 5px;
}

.low-priority {
	color: #28a745; /* Green */
}

.medium-priority {
	color: #ffc107; /* Yellow */
}

.high-priority {
	color: #dc3545; /* Red */
}

.due-date {
	font-size: 0.9rem;
	color: #6c757d;
	margin-top: 5px;
}

.task-footer {
	display: flex;
	justify-content: space-between;
	align-items: center;
	margin-top: 10px;
}

.progress-bar {
	width: 100%;
	background-color: #e9ecef;
	border-radius: 4px;
	overflow: hidden;
	height: 8px;
	margin-left: 10px;
}

.progress {
	height: 100%;
	background-color: #28a745;
	transition: width 0.3s ease;
}

.delete-task-btn {
	background: none;
	border: none;
	font-size: 1.2rem;
	cursor: pointer;
	color: inherit;
	transition: color 0.3s ease;
}

.delete-task-btn:hover {
	color: #dc3545;
}

.add-task-btn {
	background: none;
	border: 2px dashed #28a745;
	border-radius: 6px;
	padding: 10px;
	color: #28a745;
	font-weight: 500;
	cursor: pointer;
	transition: background-color 0.3s ease, transform 0.3s ease, color 0.3s ease;
}

.add-task-btn:hover {
	background-color: #28a745;
	color: #fff;
	transform: scale(1.02);
}

.kanban-board.dark .add-task-btn {
	border-color: #28a745;
	color: #28a745;
}

.kanban-board.dark .add-task-btn:hover {
	background-color: #28a745;
	color: #fff;
}

/* Inputs and Selects */
input,
textarea,
select {
	background-color: #f1f3f5;
	border: none;
	border-radius: 4px;
	padding: 10px;
	color: #212529;
}

input:focus,
textarea:focus,
select:focus {
	outline: none;
	box-shadow: 0 0 0 2px rgba(40, 167, 69, 0.25);
}

.kanban-board.dark input,
.kanban-board.dark textarea,
.kanban-board.dark select {
	background-color: #3a3a3a;
	color: #f8f9fa;
}

.kanban-board.dark input:focus,
.kanban-board.dark textarea:focus,
.kanban-board.dark select:focus {
	box-shadow: 0 0 0 2px rgba(40, 167, 69, 0.5);
}

/* Modal */
.modal-overlay {
	position: fixed;
	top: 0;
	left: 0;
	width: 100%;
	height: 100%;
	background-color: rgba(0, 0, 0, 0.5);
	display: flex;
	justify-content: center;
	align-items: center;
	opacity: 0;
	animation: fadeIn 0.3s forwards;
	z-index: 1000;
}

.modal {
	background-color: #fff;
	border-radius: 8px;
	width: 90%;
	max-width: 800px;
	max-height: 90%;
	overflow-y: auto;
	transform: scale(0.95);
	animation: slideUp 0.3s forwards;
}

.kanban-board.dark .modal {
	background-color: #2a2a2a;
	color: #f8f9fa;
}

.modal-header {
	padding: 20px;
	border-bottom: 1px solid #dee2e6;
	position: relative;
}

.modal-header h2 {
	margin: 0;
	font-size: 2rem;
}

.modal-title-input {
	width: 100%;
	font-size: 2rem;
	font-weight: bold;
	border: none;
	outline: none;
	background: transparent;
	color: inherit;
}

.modal-body {
	padding: 20px;
}

.modal-content {
	display: flex;
}

.modal-left,
.modal-right {
	flex: 1;
	padding: 10px;
}

.modal-section {
	margin-bottom: 20px;
}

.modal-section h3 {
	margin-bottom: 10px;
}

.modal-description {
	width: 100%;
	min-height: 100px;
	resize: vertical;
}

.modal-field {
	margin-bottom: 15px;
}

.modal-field label {
	display: block;
	margin-bottom: 5px;
	font-weight: 500;
}

.tags-input {
	display: flex;
	flex-wrap: wrap;
	align-items: center;
}

.tags-input .tag {
	margin-right: 5px;
	margin-bottom: 5px;
	position: relative;
}

.remove-tag {
	margin-left: 5px;
	cursor: pointer;
}

.modal-actions {
	display: flex;
	justify-content: flex-end;
	margin-top: 20px;
}

.modal-actions button {
	background-color: #28a745;
	border: none;
	padding: 10px 20px;
	color: #fff;
	font-weight: 500;
	cursor: pointer;
	border-radius: 4px;
	margin-left: 10px;
	transition: background-color 0.3s ease, transform 0.3s ease;
}

.modal-actions button:hover {
	background-color: #218838;
	transform: scale(1.03);
}

.close-modal-btn {
	position: absolute;
	top: 20px;
	right: 20px;
	background: none;
	border: none;
	font-size: 1.5rem;
	cursor: pointer;
	color: inherit;
}

/* Attachments */
.attachments-list {
	display: flex;
	flex-wrap: wrap;
}

.attachment {
	margin-right: 10px;
	margin-bottom: 10px;
	position: relative;
}

.attachment-image {
	width: 100px;
	height: 100px;
	object-fit: cover;
	border-radius: 4px;
}

/* Subtasks */
.subtasks-list {
	list-style: none;
	padding: 0;
}

.subtasks-list li {
	display: flex;
	align-items: center;
	margin-bottom: 5px;
}

.subtask-input {
	flex: 1;
	margin-left: 10px;
}

.delete-subtask-btn {
	background: none;
	border: none;
	font-size: 1rem;
	cursor: pointer;
	color: #dc3545;
}

.add-subtask-btn {
	background: none;
	border: 2px dashed #28a745;
	border-radius: 4px;
	padding: 5px 10px;
	color: #28a745;
	font-weight: 500;
	cursor: pointer;
	transition: background-color 0.3s ease, transform 0.3s ease, color 0.3s ease;
	margin-top: 10px;
}

.add-subtask-btn:hover {
	background-color: #28a745;
	color: #fff;
	transform: scale(1.02);
}

/* Chat Section */
.chat-section {
	margin-top: 20px;
}

.chat-container {
	max-height: 300px;
	overflow-y: auto;
	margin-bottom: 15px;
	display: flex;
	flex-direction: column;
}

.chat-message {
	display: flex;
	margin-bottom: 10px;
}

.chat-message.own-message {
	flex-direction: row-reverse;
}

.chat-avatar img {
	width: 40px;
	height: 40px;
	border-radius: 50%;
}

.chat-bubble {
	max-width: 70%;
	padding: 10px;
	border-radius: 10px;
	background-color: #e9ecef;
	margin: 0 10px;
	position: relative;
}

.kanban-board.dark .chat-bubble {
	background-color: #444;
	color: #f8f9fa;
}

.chat-message.own-message .chat-bubble {
	background-color: #28a745;
	color: #fff;
}

.chat-author {
	font-weight: bold;
	margin-bottom: 5px;
}

.chat-text {
	margin-bottom: 5px;
}

.chat-time {
	font-size: 0.8rem;
	color: #6c757d;
}

.chat-input {
	display: flex;
	align-items: center;
}

.chat-input textarea {
	flex: 1;
	padding: 8px;
	height: 60px;
	margin-right: 10px;
	border: none;
	border-radius: 4px;
	background-color: #f1f3f5;
}

.kanban-board.dark .chat-input textarea {
	background-color: #3a3a3a;
	color: #f8f9fa;
}

.chat-input button {
	background-color: #28a745;
	border: none;
	padding: 10px;
	color: #fff;
	font-size: 1.2rem;
	cursor: pointer;
	border-radius: 50%;
	transition: background-color 0.3s ease, transform 0.3s ease;
}

.chat-input button:hover {
	background-color: #218838;
	transform: scale(1.05);
}

/* Activity Log */
.activity-log {
	list-style: none;
	padding: 0;
}

.activity-log li {
	margin-bottom: 5px;
	font-size: 0.9rem;
	color: #6c757d;
}

/* Profile Settings Modal */
.avatar-preview {
	margin-top: 10px;
}

.avatar-preview img {
	width: 80px;
	height: 80px;
	border-radius: 50%;
}

/* Login Screen */
.login-screen {
	position: fixed;
	top: 0;
	left: 0;
	width: 100%;
	height: 100%;
	background-color: #000000a0;
	display: flex;
	justify-content: center;
	align-items: center;
	z-index: 2000;
}

.login-container {
	background-color: #fff;
	padding: 40px;
	border-radius: 8px;
	text-align: center;
}

.kanban-board.dark .login-container {
	background-color: #2a2a2a;
	color: #f8f9fa;
}

.login-container h2 {
	margin-bottom: 20px;
}

.login-container input {
	width: 100%;
	margin-bottom: 20px;
}

.login-container button {
	width: 100%;
}

.login-container input,
.login-container button {
	background-color: #f1f3f5;
	color: #212529;
}

.kanban-board.dark .login-container input,
.kanban-board.dark .login-container button {
	background-color: #3a3a3a;
	color: #f8f9fa;
}

/* Footer */
.main-footer {
	padding: 10px 20px;
	background-color: #fff;
	border-top: 1px solid #dee2e6;
	text-align: center;
}

.main-footer p {
	margin: 5px 0;
}

.footer-links {
	display: flex;
	justify-content: center;
	margin-top: 5px;
}

.footer-links a {
	margin: 0 10px;
	color: #28a745;
	text-decoration: none;
}

.footer-links a:hover {
	text-decoration: underline;
}

.kanban-board.dark .main-footer {
	background-color: #2a2a2a;
	border-top-color: #444;
}

.kanban-board.dark .footer-links a {
	color: #28a745;
}

/* Animations */
@keyframes fadeIn {
	to {
		opacity: 1;
	}
}

@keyframes slideUp {
	to {
		transform: scale(1);
	}
}

.task-transition-enter-active,
.task-transition-leave-active {
	transition: all 0.3s ease;
}

.task-transition-enter-from,
.task-transition-leave-to {
	opacity: 0;
	transform: translateY(15px);
}
</style>