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
    <div class="main-container">
      <!-- Overview Panel -->
      <div class="overview-panel" v-if="selectedTicket">
        <div class="overview-header">
          <h2>{{ selectedTicket.title }}</h2>
          <button @click="deleteTicket(selectedTicket.id)" class="btn-icon delete-btn">
            <font-awesome-icon icon="times" />
          </button>
        </div>
        <div class="overview-content">
          <div class="form-group">
            <label>Title</label>
            <input v-model="selectedTicket.title" :disabled="!isEditing" />
          </div>
          <div class="form-group">
            <label>Description</label>
            <textarea v-model="selectedTicket.description" :disabled="!isEditing"></textarea>
          </div>
          <div class="form-group">
            <label>Assignee</label>
            <select
              v-model="selectedTicket.assignee"
              :disabled="!isEditing"
              @change="updateAssigneeImagePanel"
            >
              <option v-for="agent in agents" :key="agent.name" :value="agent.name">
                {{ agent.name }}
              </option>
            </select>
          </div>
          <div class="form-group">
            <label>Assignee Image URL</label>
            <input
              v-model="selectedTicket.assigneeImage"
              :disabled="!isEditing"
              @input="validateImageURL('panel')"
            />
            <div v-if="assigneeImageError" class="error-message">
              Invalid image URL.
            </div>
          </div>
          <div class="form-group">
            <label>Priority</label>
            <select v-model="selectedTicket.priority" :disabled="!isEditing">
              <option value="Low">Low</option>
              <option value="Medium">Medium</option>
              <option value="High">High</option>
            </select>
          </div>
          <div class="form-group">
            <label>Due Date</label>
            <input type="date" v-model="selectedTicket.dueDate" :disabled="!isEditing" />
          </div>
          <div class="overview-buttons">
            <button v-if="!isEditing" @click="enableEditing" class="btn-edit">Edit</button>
            <div v-else>
              <button @click="saveTicket" class="btn-save">Save</button>
              <button @click="cancelEdit" class="btn-cancel">Cancel</button>
            </div>
          </div>
        </div>
      </div>

      <!-- Ticket Container -->
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
              @click="selectTicket(ticket)"
              :class="{ 'selected-ticket': ticket.id === selectedTicket?.id }"
            >
              <div class="ticket-header">
                <h3>{{ ticket.title }}</h3>
                <div class="ticket-actions">
                  <!-- Removed edit button -->
                  <button @click.stop="deleteTicket(ticket.id)" class="btn-icon">
                    <font-awesome-icon icon="times" />
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

      <!-- Chat Panel -->
      <div class="chat-panel" v-if="selectedTicket">
        <div class="chat-header">
          <h2>{{ selectedTicket.title }}</h2>
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
        {
          id: 2,
          title: 'Update Documentation',
          description: 'Add documentation for the new API endpoints.',
          status: 'In Progress',
          assignee: 'Jane Smith',
          assigneeImage: 'https://randomuser.me/api/portraits/women/2.jpg',
          priority: 'Medium',
          dueDate: '2023-12-05',
          chatHistory: [
            { sender: 'Jane Smith', text: 'Documentation is 50% done.', time: '2:15 PM' },
          ],
        },
        {
          id: 3,
          title: 'Server Upgrade',
          description: 'Upgrade the server to the latest version.',
          status: 'To Do',
          assignee: 'Alice Johnson',
          assigneeImage: 'https://randomuser.me/api/portraits/women/3.jpg',
          priority: 'High',
          dueDate: '2023-12-10',
          chatHistory: [
            { sender: 'Alice Johnson', text: 'Planning the upgrade schedule.', time: '9:00 AM' },
          ],
        },
        {
          id: 4,
          title: 'Design New Logo',
          description: 'Create a new logo for the company.',
          status: 'Done',
          assignee: 'Bob Brown',
          assigneeImage: 'https://randomuser.me/api/portraits/men/4.jpg',
          priority: 'Low',
          dueDate: '2023-11-20',
          chatHistory: [
            { sender: 'Bob Brown', text: 'Logo designs submitted.', time: '11:30 AM' },
            { sender: 'John Doe', text: 'Great work!', time: '12:00 PM' },
          ],
        },
      ],
      newTicketTitle: '',
      newTicketDescription: '',
      newTicketAssignee: '',
      newTicketAssigneeImage: '',
      newAssigneeImageError: false,
      newTicketPriority: 'Low',
      newTicketDueDate: '',
      newTicketStatus: '',
      isEditing: false,
      assigneeImageError: false,
      ticketStatuses: ['To Do', 'In Progress', 'Done'],
      draggedTicket: null,
      agents: [
        { name: 'John Doe', image: 'https://randomuser.me/api/portraits/men/1.jpg' },
        { name: 'Jane Smith', image: 'https://randomuser.me/api/portraits/women/2.jpg' },
        { name: 'Alice Johnson', image: 'https://randomuser.me/api/portraits/women/3.jpg' },
        { name: 'Bob Brown', image: 'https://randomuser.me/api/portraits/men/4.jpg' },
      ],
      selectedTicket: null,
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
      if (this.selectedTicket && this.selectedTicket.id === id) {
        this.selectedTicket = null;
      }
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
    selectTicket(ticket) {
      this.selectedTicket = ticket;
      this.isEditing = false; // Reset editing state
      this.assigneeImageError = false;
      if (!this.selectedTicket.chatHistory) {
        this.$set(this.selectedTicket, 'chatHistory', []);
      }
      this.$nextTick(() => {
        const messagesContainer = this.$el.querySelector('.messages');
        if (messagesContainer) {
          messagesContainer.scrollTop = messagesContainer.scrollHeight;
        }
      });
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
          if (messagesContainer) {
            messagesContainer.scrollTop = messagesContainer.scrollHeight;
          }
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
    updateAssigneeImagePanel() {
      const agent = this.agents.find((agent) => agent.name === this.selectedTicket.assignee);
      if (agent) {
        this.selectedTicket.assigneeImage = agent.image;
        this.assigneeImageError = false;
      }
    },
    validateImageURL(type) {
      let url;
      if (type === 'new') {
        url = this.newTicketAssigneeImage;
      } else if (type === 'panel') {
        url = this.selectedTicket.assigneeImage;
      } else {
        return;
      }
      const img = new Image();
      img.onload = () => {
        if (type === 'new') {
          this.newAssigneeImageError = false;
        } else if (type === 'panel') {
          this.assigneeImageError = false;
        }
      };
      img.onerror = () => {
        if (type === 'new') {
          this.newAssigneeImageError = true;
        } else if (type === 'panel') {
          this.assigneeImageError = true;
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
    enableEditing() {
      this.isEditing = true;
    },
    saveTicket() {
      if (this.assigneeImageError) {
        return;
      }
      this.isEditing = false;
    },
    cancelEdit() {
      // Revert changes by reloading the selected ticket
      if (this.selectedTicket) {
        const ticket = this.tickets.find((t) => t.id === this.selectedTicket.id);
        if (ticket) {
          // Reassign the properties
          this.selectedTicket.title = ticket.title;
          this.selectedTicket.description = ticket.description;
          this.selectedTicket.assignee = ticket.assignee;
          this.selectedTicket.assigneeImage = ticket.assigneeImage;
          this.selectedTicket.priority = ticket.priority;
          this.selectedTicket.dueDate = ticket.dueDate;
        }
      }
      this.isEditing = false;
      this.assigneeImageError = false;
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
  max-width: 1400px;
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

/* Main container to hold overview, tickets, and chat panel */
.main-container {
  display: flex;
  gap: 20px;
}

/* Overview Panel Styles */
.overview-panel {
  flex: 1;
  background-color: var(--ticket-column-background);
  padding: 20px;
  border-radius: 12px;
  border: 1px solid var(--border-color);
  display: flex;
  flex-direction: column;
  transition: background-color 0.5s ease;
}
.overview-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
}
.overview-header h2 {
  margin: 0;
  color: var(--header-color);
}
.delete-btn {
  font-size: 1.2rem;
}
.overview-content {
  margin-top: 20px;
}
.form-group {
  margin-bottom: 15px;
}
.form-group label {
  display: block;
  margin-bottom: 5px;
  font-weight: 500;
}
.overview-content input,
.overview-content textarea,
.overview-content select {
  width: 100%;
  padding: 10px;
  border-radius: 8px;
  border: 1px solid var(--border-color);
  background-color: var(--input-background);
  color: var(--input-text-color);
  outline: none;
  transition: background-color 0.5s ease, color 0.5s ease;
}
.overview-buttons {
  display: flex;
  gap: 10px;
  margin-top: 20px;
}
.btn-edit,
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
.btn-edit:hover,
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

/* Ticket Container Styles */
.ticket-container {
  flex: 2;
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

/* Chat Panel Styles */
.chat-panel {
  flex: 1;
  background-color: var(--ticket-column-background);
  border: 1px solid var(--border-color);
  border-radius: 12px;
  overflow: hidden;
  display: flex;
  flex-direction: column;
  transition: background-color 0.5s ease;
}
.chat-header {
  background-color: var(--navbar-background);
  padding: 15px;
  border-bottom: 1px solid var(--border-color);
}
.chat-header h2 {
  margin: 0;
  color: var(--header-color);
}
.messages {
  flex: 1;
  overflow-y: auto;
  padding: 15px;
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
  padding: 10px 15px;
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

/* Selected Ticket Highlight */
.selected-ticket {
  border: 2px solid var(--header-color);
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

/* Error Message */
.error-message {
  color: #e74c3c;
  font-size: 0.9rem;
  margin-top: 5px;
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
