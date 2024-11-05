<template>
  <div id="app" :class="{ dark: isDarkMode, light: !isDarkMode }">
    <header>
      <h1>IT Ticket System</h1>
    </header>
    <nav class="navbar">
      <div class="navbar-left">
        <ul>
          <li><a href="#">Home</a></li>
          <!-- Placeholder for future extras -->
          <li><a href="#">Features</a></li>
          <li><a href="#">Settings</a></li>
        </ul>
        <div class="search-container">
          <input
            type="text"
            v-model="searchQuery"
            placeholder="Search tickets..."
            class="search-input"
          />
          <font-awesome-icon icon="search" class="search-icon" />
        </div>
      </div>
      <button @click="toggleDarkMode" class="btn-toggle-mode">
        Switch to {{ isDarkMode ? 'Light' : 'Dark' }} Mode
      </button>
    </nav>
    <div class="ticket-container">
      <div
        v-for="status in ticketStatuses"
        :key="status"
        class="ticket-column"
        @dragover.prevent
        @drop="onDrop($event, status)"
      >
        <h2>{{ status }}</h2>
        <transition-group name="ticket" tag="div">
          <div
            v-for="ticket in filteredTickets(status)"
            :key="ticket.id"
            class="ticket-card"
            draggable="true"
            @dragstart="onDragStart($event, ticket)"
            @click="openChat(ticket)"
          >
            <div class="ticket-header">
              <h3>{{ ticket.title }}</h3>
              <div class="ticket-actions">
                <button @click.stop="editTicket(ticket)" class="btn-icon">
                  <font-awesome-icon icon="edit" />
                </button>
                <button @click.stop="deleteTicket(ticket.id)" class="btn-icon">
                  <font-awesome-icon icon="trash" />
                </button>
              </div>
            </div>
            <p class="ticket-description">{{ ticket.description }}</p>
            <div class="ticket-info">
              <div class="assignee-info">
                <img :src="ticket.assigneeImage" alt="Assignee Image" class="assignee-image" />
                <p>{{ ticket.assignee }}</p>
              </div>
              <div class="ticket-meta">
                <span class="priority" :class="ticket.priority.toLowerCase()">
                  {{ ticket.priority }} Priority
                </span>
                <span class="due-date">Due: {{ ticket.dueDate }}</span>
              </div>
            </div>
          </div>
          <!-- Placeholder Ticket -->
          <div
            class="ticket-card placeholder-ticket"
            @click="openAddTicketModal(status)"
            key="placeholder"
          >
            <h3>+ Add New Ticket</h3>
          </div>
        </transition-group>
      </div>
    </div>

    <!-- Add Ticket Modal -->
    <transition name="modal">
      <div v-if="showAddTicketModal" class="modal-overlay">
        <div class="modal-content">
          <h2>Add New Ticket in {{ newTicketStatus }}</h2>
          <div class="form-group">
            <label>Title</label>
            <input v-model="newTicketTitle" placeholder="Ticket Title" />
          </div>
          <div class="form-group">
            <label>Description</label>
            <textarea v-model="newTicketDescription" placeholder="Ticket Description"></textarea>
          </div>
          <div class="form-group">
            <label>Assignee</label>
            <select v-model="newTicketAssignee" @change="updateAssigneeImage">
              <option value="" disabled>Select Assignee</option>
              <option v-for="agent in agents" :key="agent.name" :value="agent.name">
                {{ agent.name }}
              </option>
            </select>
          </div>
          <div class="form-group">
            <label>Assignee Image URL</label>
            <input
              v-model="newTicketAssigneeImage"
              placeholder="Assignee Image URL"
              @input="validateImageURL('new')"
            />
            <div v-if="newAssigneeImageError" class="error-message">
              Invalid image URL.
            </div>
          </div>
          <div class="form-group">
            <label>Priority</label>
            <select v-model="newTicketPriority">
              <option value="Low">Low</option>
              <option value="Medium">Medium</option>
              <option value="High">High</option>
            </select>
          </div>
          <div class="form-group">
            <label>Due Date</label>
            <input type="date" v-model="newTicketDueDate" />
          </div>
          <div class="modal-buttons">
            <button @click="addTicket" class="btn-add">Add Ticket</button>
            <button @click="closeAddTicketModal" class="btn-cancel">Cancel</button>
          </div>
        </div>
      </div>
    </transition>

    <!-- Edit Ticket Modal -->
    <transition name="modal">
      <div v-if="isEditing" class="modal-overlay">
        <div class="modal-content">
          <h2>Edit Ticket</h2>
          <div class="form-group">
            <label>Title</label>
            <input v-model="editTicketTitle" placeholder="Ticket Title" />
          </div>
          <div class="form-group">
            <label>Description</label>
            <textarea v-model="editTicketDescription" placeholder="Ticket Description"></textarea>
          </div>
          <div class="form-group">
            <label>Assignee</label>
            <select v-model="editTicketAssignee" @change="updateEditAssigneeImage">
              <option value="" disabled>Select Assignee</option>
              <option v-for="agent in agents" :key="agent.name" :value="agent.name">
                {{ agent.name }}
              </option>
            </select>
          </div>
          <div class="form-group">
            <label>Assignee Image URL</label>
            <input
              v-model="editTicketAssigneeImage"
              placeholder="Assignee Image URL"
              @input="validateImageURL('edit')"
            />
            <div v-if="editAssigneeImageError" class="error-message">
              Invalid image URL.
            </div>
          </div>
          <div class="form-group">
            <label>Priority</label>
            <select v-model="editTicketPriority">
              <option value="Low">Low</option>
              <option value="Medium">Medium</option>
              <option value="High">High</option>
            </select>
          </div>
          <div class="form-group">
            <label>Due Date</label>
            <input type="date" v-model="editTicketDueDate" />
          </div>
          <div class="modal-buttons">
            <button @click="saveTicket" class="btn-save">Save Changes</button>
            <button @click="cancelEdit" class="btn-cancel">Cancel</button>
          </div>
        </div>
      </div>
    </transition>

    <!-- Chat History Modal -->
    <transition name="modal">
      <div v-if="showChat" class="chat-history">
        <div class="chat-header">
          <h2>{{ selectedTicket.title }}</h2>
          <button @click="closeChat" class="btn-icon close-chat-btn">
            <font-awesome-icon icon="times" />
          </button>
        </div>
        <div class="messages">
          <div
            v-for="(message, index) in selectedTicket.chatHistory"
            :key="index"
            :class="['message', message.sender === 'You' ? 'sent' : 'received']"
          >
            <img
              :src="getSenderImage(message.sender)"
              alt="Sender Image"
              class="message-image"
            />
            <div class="message-content">
              <p>{{ message.text }}</p>
              <span class="message-time">{{ message.time }}</span>
            </div>
          </div>
        </div>
        <div class="new-message">
          <input
            v-model="newMessageText"
            placeholder="Type a message..."
            @keyup.enter="sendMessage"
          />
          <button @click="sendMessage" class="btn-icon">
            <font-awesome-icon icon="paper-plane" />
          </button>
        </div>
      </div>
    </transition>
  </div>
</template>

<script>
import { library } from '@fortawesome/fontawesome-svg-core';
import {
  faEdit,
  faTrash,
  faPaperPlane,
  faTimes,
  faSearch,
} from '@fortawesome/free-solid-svg-icons';
import { FontAwesomeIcon } from '@fortawesome/vue-fontawesome';

library.add(faEdit, faTrash, faPaperPlane, faTimes, faSearch);

export default {
  name: 'App',
  components: {
    FontAwesomeIcon,
  },
  data() {
    return {
      tickets: [
        {
          id: 1,
          title: 'Bug in Login Page',
          description: 'The login page throws an error.',
          status: 'To Do',
          assignee: 'John Doe',
          assigneeImage: 'https://randomuser.me/api/portraits/men/1.jpg',
          priority: 'High',
          dueDate: '2023-12-01',
          chatHistory: [
            { sender: 'John Doe', text: 'We need to fix this ASAP.', time: '10:00 AM' },
            { sender: 'Jane Smith', text: "I'm on it.", time: '10:05 AM' },
          ],
        },
        // ... (other tickets)
      ],
      newTicketTitle: '',
      newTicketDescription: '',
      newTicketAssignee: '',
      newTicketAssigneeImage: '',
      newAssigneeImageError: false,
      newTicketPriority: 'Low',
      newTicketDueDate: '',
      newTicketStatus: '',
      editTicketId: null,
      editTicketTitle: '',
      editTicketDescription: '',
      editTicketAssignee: '',
      editTicketAssigneeImage: '',
      editAssigneeImageError: false,
      editTicketPriority: 'Low',
      editTicketDueDate: '',
      isEditing: false,
      ticketStatuses: ['To Do', 'In Progress', 'Done'],
      draggedTicket: null,
      agents: [
        { name: 'John Doe', image: 'https://randomuser.me/api/portraits/men/1.jpg' },
        { name: 'Jane Smith', image: 'https://randomuser.me/api/portraits/women/2.jpg' },
        { name: 'Alice Johnson', image: 'https://randomuser.me/api/portraits/women/3.jpg' },
        { name: 'Bob Brown', image: 'https://randomuser.me/api/portraits/men/4.jpg' },
      ],
      selectedTicket: null,
      showChat: false,
      newMessageText: '',
      isDarkMode: true,
      showAddTicketModal: false,
      searchQuery: '',
    };
  },
  computed: {
    filteredTickets() {
      return (status) => {
        return this.tickets.filter((ticket) => {
          const matchesStatus = ticket.status === status;
          const matchesSearch =
            ticket.title.toLowerCase().includes(this.searchQuery.toLowerCase()) ||
            ticket.description.toLowerCase().includes(this.searchQuery.toLowerCase()) ||
            ticket.assignee.toLowerCase().includes(this.searchQuery.toLowerCase());
          return matchesStatus && matchesSearch;
        });
      };
    },
  },
  methods: {
    addTicket() {
      if (
        this.newTicketTitle &&
        this.newTicketDescription &&
        this.newTicketAssignee &&
        this.newTicketDueDate &&
        !this.newAssigneeImageError
      ) {
        this.tickets.push({
          id: this.tickets.length + 1,
          title: this.newTicketTitle,
          description: this.newTicketDescription,
          status: this.newTicketStatus || 'To Do',
          assignee: this.newTicketAssignee,
          assigneeImage: this.newTicketAssigneeImage,
          priority: this.newTicketPriority,
          dueDate: this.newTicketDueDate,
          chatHistory: [],
        });
        // Reset fields
        this.newTicketTitle = '';
        this.newTicketDescription = '';
        this.newTicketAssignee = '';
        this.newTicketAssigneeImage = '';
        this.newTicketPriority = 'Low';
        this.newTicketDueDate = '';
        this.newTicketStatus = '';
        this.newAssigneeImageError = false;
        this.showAddTicketModal = false;
      }
    },
    deleteTicket(id) {
      this.tickets = this.tickets.filter((ticket) => ticket.id !== id);
    },
    editTicket(ticket) {
      this.isEditing = true;
      this.editTicketId = ticket.id;
      this.editTicketTitle = ticket.title;
      this.editTicketDescription = ticket.description;
      this.editTicketAssignee = ticket.assignee;
      this.editTicketAssigneeImage = ticket.assigneeImage;
      this.editTicketPriority = ticket.priority;
      this.editTicketDueDate = ticket.dueDate;
    },
    saveTicket() {
      if (!this.editAssigneeImageError) {
        const ticket = this.tickets.find((ticket) => ticket.id === this.editTicketId);
        if (ticket) {
          ticket.title = this.editTicketTitle;
          ticket.description = this.editTicketDescription;
          ticket.assignee = this.editTicketAssignee;
          ticket.assigneeImage = this.editTicketAssigneeImage;
          ticket.priority = this.editTicketPriority;
          ticket.dueDate = this.editTicketDueDate;
        }
        this.cancelEdit();
      }
    },
    cancelEdit() {
      this.isEditing = false;
      this.editTicketId = null;
      this.editTicketTitle = '';
      this.editTicketDescription = '';
      this.editTicketAssignee = '';
      this.editTicketAssigneeImage = '';
      this.editTicketPriority = 'Low';
      this.editTicketDueDate = '';
      this.editAssigneeImageError = false;
    },
    onDragStart(event, ticket) {
      this.draggedTicket = ticket;
    },
    onDrop(event, status) {
      if (this.draggedTicket) {
        this.draggedTicket.status = status;
        this.draggedTicket = null;
      }
    },
    openChat(ticket) {
      this.selectedTicket = ticket;
      this.showChat = true;
      if (!this.selectedTicket.chatHistory) {
        this.$set(this.selectedTicket, 'chatHistory', []);
      }
      this.$nextTick(() => {
        const messagesContainer = this.$el.querySelector('.messages');
        messagesContainer.scrollTop = messagesContainer.scrollHeight;
      });
    },
    closeChat() {
      this.showChat = false;
      this.selectedTicket = null;
      this.newMessageText = '';
    },
    sendMessage() {
      if (this.newMessageText.trim() !== '') {
        const time = new Date().toLocaleTimeString([], { hour: '2-digit', minute: '2-digit' });
        this.selectedTicket.chatHistory.push({
          sender: 'You',
          text: this.newMessageText.trim(),
          time,
        });
        this.newMessageText = '';
        this.$nextTick(() => {
          const messagesContainer = this.$el.querySelector('.messages');
          messagesContainer.scrollTop = messagesContainer.scrollHeight;
        });
      }
    },
    toggleDarkMode() {
      this.isDarkMode = !this.isDarkMode;
    },
    openAddTicketModal(status) {
      this.showAddTicketModal = true;
      this.newTicketStatus = status;
    },
    closeAddTicketModal() {
      this.showAddTicketModal = false;
      // Reset fields
      this.newTicketTitle = '';
      this.newTicketDescription = '';
      this.newTicketAssignee = '';
      this.newTicketAssigneeImage = '';
      this.newTicketPriority = 'Low';
      this.newTicketDueDate = '';
      this.newTicketStatus = '';
      this.newAssigneeImageError = false;
    },
    updateAssigneeImage() {
      const agent = this.agents.find((agent) => agent.name === this.newTicketAssignee);
      if (agent) {
        this.newTicketAssigneeImage = agent.image;
        this.newAssigneeImageError = false;
      }
    },
    updateEditAssigneeImage() {
      const agent = this.agents.find((agent) => agent.name === this.editTicketAssignee);
      if (agent) {
        this.editTicketAssigneeImage = agent.image;
        this.editAssigneeImageError = false;
      }
    },
    validateImageURL(type) {
      const url = type === 'new' ? this.newTicketAssigneeImage : this.editTicketAssigneeImage;
      const img = new Image();
      img.onload = () => {
        if (type === 'new') {
          this.newAssigneeImageError = false;
        } else {
          this.editAssigneeImageError = false;
        }
      };
      img.onerror = () => {
        if (type === 'new') {
          this.newAssigneeImageError = true;
        } else {
          this.editAssigneeImageError = true;
        }
      };
      img.src = url;
    },
    getSenderImage(senderName) {
      if (senderName === 'You') {
        return 'https://i.pravatar.cc/150?img=5'; // Placeholder image for "You"
      } else {
        const agent = this.agents.find((agent) => agent.name === senderName);
        return agent ? agent.image : 'https://via.placeholder.com/150';
      }
    },
  },
};
</script>

<style scoped>
@import url("https://fonts.googleapis.com/css2?family=Roboto:wght@400;500;700&display=swap");

:root {
  --background-color: #ffffff;
  --text-color: #000000;
  --header-color: #00d1b2;
  --navbar-background: #f8f8f8;
  --ticket-column-background: #eaeaea;
  --ticket-card-background: #ffffff;
  --input-background: #ffffff;
  --input-text-color: #000000;
  --border-color: #cccccc;
}

.dark {
  --background-color: #121212;
  --text-color: #ffffff;
  --header-color: #00d1b2;
  --navbar-background: #1f1f1f;
  --ticket-column-background: #1f1f1f;
  --ticket-card-background: #282828;
  --input-background: #333333;
  --input-text-color: #ffffff;
  --border-color: #2c2c2c;
}

.light {
  --background-color: #ffffff;
  --text-color: #000000;
  --header-color: #00d1b2;
  --navbar-background: #f8f8f8;
  --ticket-column-background: #eaeaea;
  --ticket-card-background: #ffffff;
  --input-background: #ffffff;
  --input-text-color: #000000;
  --border-color: #cccccc;
}

body {
  margin: 0;
  padding: 0;
}

#app {
  font-family: "Roboto", sans-serif;
  max-width: 1200px;
  margin: 0 auto;
  padding: 20px;
  color: var(--text-color);
  background-color: var(--background-color);
  border-radius: 12px;
  box-shadow: 0 4px 15px rgba(0, 0, 0, 0.1);
  transition: background-color 0.5s ease, color 0.5s ease;
}
header {
  text-align: center;
  margin-bottom: 20px;
}
header h1 {
  font-size: 2.5rem;
  font-weight: 700;
  color: var(--header-color);
}

/* Navbar Styles */
.navbar {
  background-color: var(--navbar-background);
  padding: 10px 20px;
  margin-bottom: 20px;
  border-radius: 12px;
  display: flex;
  justify-content: space-between;
  align-items: center;
  transition: background-color 0.5s ease;
}
.navbar-left {
  display: flex;
  align-items: center;
}
.navbar ul {
  list-style-type: none;
  display: flex;
  gap: 15px;
  margin: 0;
  padding: 0;
}
.navbar ul li a {
  color: var(--header-color);
  text-decoration: none;
  font-weight: 500;
  transition: color 0.3s;
}
.navbar ul li a:hover {
  color: #00bfa5;
}
.search-container {
  position: relative;
  margin-left: 20px;
}
.search-input {
  padding: 8px 35px 8px 15px;
  border-radius: 20px;
  border: 1px solid var(--border-color);
  background-color: var(--input-background);
  color: var(--input-text-color);
  outline: none;
}
.search-icon {
  position: absolute;
  right: 10px;
  top: 50%;
  transform: translateY(-50%);
  color: var(--text-color);
}
.btn-toggle-mode {
  padding: 8px 15px;
  background-color: var(--header-color);
  color: #ffffff;
  border: none;
  border-radius: 8px;
  cursor: pointer;
}
.btn-toggle-mode:hover {
  background-color: #00bfa5;
}

.ticket-container {
  display: flex;
  flex-wrap: wrap;
  justify-content: space-between;
  gap: 20px;
}
.ticket-column {
  flex: 1;
  min-width: 300px;
  background: var(--ticket-column-background);
  padding: 20px;
  border-radius: 12px;
  border: 1px solid var(--border-color);
  box-shadow: 0 4px 10px rgba(0, 0, 0, 0.1);
  transition: background-color 0.5s ease;
}
.ticket-column h2 {
  font-size: 1.8rem;
  margin-bottom: 15px;
  color: var(--header-color);
}
.ticket-card {
  background: var(--ticket-card-background);
  padding: 15px;
  margin-bottom: 15px;
  border-radius: 12px;
  box-shadow: 0 4px 10px rgba(0, 0, 0, 0.1);
  transition: transform 0.2s, background-color 0.5s ease;
  cursor: grab;
  border: 1px solid var(--border-color);
}
.ticket-card:hover {
  transform: scale(1.02);
}
.ticket-card:active {
  cursor: grabbing;
}
.ticket-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
}
.ticket-header h3 {
  margin: 0;
  font-weight: 500;
  color: var(--text-color);
}
.ticket-description {
  margin: 10px 0;
  color: var(--text-color);
}
.ticket-info {
  display: flex;
  justify-content: space-between;
  align-items: center;
}
.assignee-info {
  display: flex;
  align-items: center;
  gap: 10px;
}
.assignee-image {
  width: 40px;
  height: 40px;
  border-radius: 50%;
  object-fit: cover;
}
.ticket-meta {
  display: flex;
  flex-direction: column;
  align-items: flex-end;
}
.priority {
  font-weight: bold;
  padding: 5px 10px;
  border-radius: 20px;
  color: #ffffff;
}
.priority.low {
  background-color: #27ae60;
}
.priority.medium {
  background-color: #f39c12;
}
.priority.high {
  background-color: #e74c3c;
}
.due-date {
  margin-top: 5px;
  font-size: 0.9rem;
  color: var(--text-color);
}
.ticket-actions {
  display: flex;
  gap: 10px;
}
.btn-icon {
  background: none;
  border: none;
  cursor: pointer;
  color: var(--text-color);
  font-size: 1.2rem;
}
.btn-icon:hover {
  color: var(--header-color);
}

/* Placeholder Ticket Styles */
.placeholder-ticket {
  background-color: transparent;
  border: 2px dashed var(--text-color);
  color: var(--text-color);
  text-align: center;
  padding: 20px;
  cursor: pointer;
  transition: background-color 0.3s;
}
.placeholder-ticket:hover {
  background-color: rgba(0, 0, 0, 0.05);
}
.placeholder-ticket h3 {
  margin: 0;
}

/* Modal Styles */
.modal-overlay {
  position: fixed;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  background-color: rgba(18, 18, 18, 0.8);
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 1000;
}
.modal-content {
  background-color: var(--ticket-column-background);
  padding: 30px;
  border-radius: 12px;
  width: 400px;
  color: var(--text-color);
  position: relative;
  transition: background-color 0.5s ease;
  box-shadow: 0 4px 15px rgba(0, 0, 0, 0.3);
}
.modal-content h2 {
  margin-bottom: 20px;
  color: var(--header-color);
  text-align: center;
}
.form-group {
  margin-bottom: 15px;
}
.form-group label {
  display: block;
  margin-bottom: 5px;
  font-weight: 500;
}
.modal-content input,
.modal-content textarea,
.modal-content select {
  width: 100%;
  padding: 10px;
  border-radius: 8px;
  border: 1px solid var(--border-color);
  background-color: var(--input-background);
  color: var(--input-text-color);
  outline: none;
  transition: background-color 0.5s ease, color 0.5s ease;
}
.modal-content input::placeholder,
.modal-content textarea::placeholder {
  color: #888;
}
.modal-buttons {
  display: flex;
  justify-content: space-between;
  margin-top: 20px;
}
.btn-add,
.btn-save,
.btn-cancel {
  padding: 10px 20px;
  background-color: var(--header-color);
  color: white;
  border: none;
  border-radius: 8px;
  cursor: pointer;
  transition: background-color 0.3s;
}
.btn-add:hover,
.btn-save:hover,
.btn-cancel:hover {
  background-color: #00bfa5;
}
.btn-cancel {
  background-color: #f39c12;
}
.btn-cancel:hover {
  background-color: #e67e22;
}

/* Error Message */
.error-message {
  color: #e74c3c;
  font-size: 0.9rem;
  margin-top: 5px;
}

/* Chat History Styles */
.chat-history {
  position: fixed;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  background-color: var(--background-color);
  display: flex;
  flex-direction: column;
  color: var(--text-color);
  z-index: 1000;
}
.chat-header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 15px 20px;
  background-color: var(--navbar-background);
  border-bottom: 1px solid var(--border-color);
}
.chat-header h2 {
  margin: 0;
  color: var(--header-color);
}
.close-chat-btn {
  font-size: 1.5rem;
  color: var(--text-color);
  background: none;
  border: none;
  cursor: pointer;
}
.close-chat-btn:hover {
  color: var(--header-color);
}
.messages {
  flex: 1;
  overflow-y: auto;
  padding: 20px;
}
.message {
  display: flex;
  align-items: flex-end;
  margin-bottom: 15px;
}
.message.sent {
  flex-direction: row-reverse;
}
.message-image {
  width: 40px;
  height: 40px;
  border-radius: 50%;
  object-fit: cover;
  margin: 0 10px;
}
.message-content {
  max-width: 70%;
  background-color: #f1f0f0;
  color: #000000;
  padding: 10px 15px;
  border-radius: 15px;
  position: relative;
  animation: fadeIn 0.3s ease-in-out;
}
.message.sent .message-content {
  background-color: #007bff;
  color: #ffffff;
}
.message-time {
  font-size: 0.8rem;
  margin-top: 5px;
  color: #888;
}
.new-message {
  display: flex;
  padding: 10px 20px;
  border-top: 1px solid var(--border-color);
}
.new-message input {
  flex: 1;
  padding: 10px;
  border-radius: 20px;
  border: 1px solid var(--border-color);
  background-color: var(--input-background);
  color: var(--input-text-color);
  transition: background-color 0.5s ease, color 0.5s ease;
}
.new-message button {
  padding: 0 15px;
  border: none;
  background: none;
  font-size: 1.5rem;
  color: var(--header-color);
  cursor: pointer;
}
.new-message button:hover {
  color: #00bfa5;
}

/* Animations */
.ticket-enter-active,
.ticket-leave-active {
  transition: all 0.5s ease;
}
.ticket-enter,
.ticket-leave-to {
  opacity: 0;
  transform: translateY(20px);
}

.modal-enter-active,
.modal-leave-active {
  transition: opacity 0.5s ease;
}
.modal-enter,
.modal-leave-to {
  opacity: 0;
}

@keyframes fadeIn {
  from {
    opacity: 0;
    transform: translateY(10px);
  }
  to {
    opacity: 1;
    transform: translateY(0);
  }
}
</style>
