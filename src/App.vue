<template>
  <div id="app" :class="{ dark: isDarkMode, light: !isDarkMode }">
    <header>
      <h1>IT Ticket System</h1>
    </header>
    <nav class="navbar">
      <ul>
        <li><a href="#">Home</a></li>
        <!-- Placeholder for future extras -->
        <li><a href="#">Features</a></li>
        <li><a href="#">Settings</a></li>
      </ul>
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
            <h3>{{ ticket.title }}</h3>
            <p>{{ ticket.description }}</p>
            <p><strong>Assignee:</strong> {{ ticket.assignee }}</p>
            <p><strong>Priority:</strong> {{ ticket.priority }}</p>
            <p><strong>Due Date:</strong> {{ ticket.dueDate }}</p>
            <div class="ticket-actions">
              <button @click.stop="editTicket(ticket)" class="btn-edit">Edit</button>
              <button @click.stop="deleteTicket(ticket.id)" class="btn-delete">Delete</button>
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
          <input v-model="newTicketTitle" placeholder="Ticket Title" />
          <textarea v-model="newTicketDescription" placeholder="Ticket Description"></textarea>
          <select v-model="newTicketAssignee">
            <option v-for="agent in agents" :key="agent" :value="agent">{{ agent }}</option>
          </select>
          <select v-model="newTicketPriority">
            <option value="Low">Low</option>
            <option value="Medium">Medium</option>
            <option value="High">High</option>
          </select>
          <input type="date" v-model="newTicketDueDate" />
          <button @click="addTicket" class="btn-add">Add Ticket</button>
          <button @click="closeAddTicketModal" class="btn-cancel">Cancel</button>
        </div>
      </div>
    </transition>

    <!-- Edit Ticket Modal -->
    <transition name="modal">
      <div v-if="isEditing" class="modal-overlay">
        <div class="modal-content">
          <h2>Edit Ticket</h2>
          <input v-model="editTicketTitle" placeholder="Ticket Title" />
          <textarea v-model="editTicketDescription" placeholder="Ticket Description"></textarea>
          <select v-model="editTicketAssignee">
            <option v-for="agent in agents" :key="agent" :value="agent">{{ agent }}</option>
          </select>
          <select v-model="editTicketPriority">
            <option value="Low">Low</option>
            <option value="Medium">Medium</option>
            <option value="High">High</option>
          </select>
          <input type="date" v-model="editTicketDueDate" />
          <button @click="saveTicket" class="btn-save">Save Changes</button>
          <button @click="cancelEdit" class="btn-cancel">Cancel</button>
        </div>
      </div>
    </transition>

    <!-- Chat History Modal -->
    <transition name="modal">
      <div v-if="showChat" class="chat-history">
        <h2>Chat History for {{ selectedTicket.title }}</h2>
        <div class="messages">
          <div
            v-for="(message, index) in selectedTicket.chatHistory"
            :key="index"
            :class="['message', message.sender === 'You' ? 'sent' : 'received']"
          >
            <div class="message-content">
              <p>{{ message.text }}</p>
            </div>
            <span class="message-sender">{{ message.sender }}</span>
          </div>
        </div>
        <div class="new-message">
          <input v-model="newMessageText" placeholder="Type a message..." @keyup.enter="sendMessage" />
          <button @click="sendMessage">Send</button>
        </div>
        <button @click="closeChat" class="btn-close-chat">Close Chat</button>
      </div>
    </transition>
  </div>
</template>

<script>
export default {
  name: "App",
  data() {
    return {
      tickets: [
        {
          id: 1,
          title: "Bug in Login Page",
          description: "The login page throws an error.",
          status: "To Do",
          assignee: "John Doe",
          priority: "High",
          dueDate: "2023-12-01",
          chatHistory: [
            { sender: "John Doe", text: "We need to fix this ASAP." },
            { sender: "Jane Smith", text: "I'm on it." },
          ],
        },
        {
          id: 2,
          title: "Update Documentation",
          description: "Add documentation for the new API endpoints.",
          status: "In Progress",
          assignee: "Jane Smith",
          priority: "Medium",
          dueDate: "2023-12-05",
          chatHistory: [{ sender: "Jane Smith", text: "Documentation is 50% done." }],
        },
        {
          id: 3,
          title: "Server Upgrade",
          description: "Upgrade the server to the latest version.",
          status: "To Do",
          assignee: "Alice Johnson",
          priority: "High",
          dueDate: "2023-12-10",
          chatHistory: [{ sender: "Alice Johnson", text: "Planning the upgrade schedule." }],
        },
        {
          id: 4,
          title: "Design New Logo",
          description: "Create a new logo for the company.",
          status: "Done",
          assignee: "Bob Brown",
          priority: "Low",
          dueDate: "2023-11-20",
          chatHistory: [
            { sender: "Bob Brown", text: "Logo designs submitted." },
            { sender: "John Doe", text: "Great work!" },
          ],
        },
      ],
      newTicketTitle: "",
      newTicketDescription: "",
      newTicketAssignee: "",
      newTicketPriority: "Low",
      newTicketDueDate: "",
      newTicketStatus: "",
      editTicketId: null,
      editTicketTitle: "",
      editTicketDescription: "",
      editTicketAssignee: "",
      editTicketPriority: "Low",
      editTicketDueDate: "",
      isEditing: false,
      ticketStatuses: ["To Do", "In Progress", "Done"],
      draggedTicket: null,
      agents: ["John Doe", "Jane Smith", "Alice Johnson", "Bob Brown"],
      selectedTicket: null,
      showChat: false,
      newMessageText: "",
      isDarkMode: true,
      showAddTicketModal: false,
    };
  },
  methods: {
    filteredTickets(status) {
      return this.tickets.filter((ticket) => ticket.status === status);
    },
    addTicket() {
      if (
        this.newTicketTitle &&
        this.newTicketDescription &&
        this.newTicketAssignee &&
        this.newTicketDueDate
      ) {
        this.tickets.push({
          id: this.tickets.length + 1,
          title: this.newTicketTitle,
          description: this.newTicketDescription,
          status: this.newTicketStatus || "To Do",
          assignee: this.newTicketAssignee,
          priority: this.newTicketPriority,
          dueDate: this.newTicketDueDate,
          chatHistory: [],
        });
        // Reset fields
        this.newTicketTitle = "";
        this.newTicketDescription = "";
        this.newTicketAssignee = "";
        this.newTicketPriority = "Low";
        this.newTicketDueDate = "";
        this.newTicketStatus = "";
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
      this.editTicketPriority = ticket.priority;
      this.editTicketDueDate = ticket.dueDate;
    },
    saveTicket() {
      const ticket = this.tickets.find((ticket) => ticket.id === this.editTicketId);
      if (ticket) {
        ticket.title = this.editTicketTitle;
        ticket.description = this.editTicketDescription;
        ticket.assignee = this.editTicketAssignee;
        ticket.priority = this.editTicketPriority;
        ticket.dueDate = this.editTicketDueDate;
      }
      this.cancelEdit();
    },
    cancelEdit() {
      this.isEditing = false;
      this.editTicketId = null;
      this.editTicketTitle = "";
      this.editTicketDescription = "";
      this.editTicketAssignee = "";
      this.editTicketPriority = "Low";
      this.editTicketDueDate = "";
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
        this.$set(this.selectedTicket, "chatHistory", []);
      }
      this.$nextTick(() => {
        const messagesContainer = this.$el.querySelector(".messages");
        messagesContainer.scrollTop = messagesContainer.scrollHeight;
      });
    },
    closeChat() {
      this.showChat = false;
      this.selectedTicket = null;
      this.newMessageText = "";
    },
    sendMessage() {
      if (this.newMessageText.trim() !== "") {
        this.selectedTicket.chatHistory.push({
          sender: "You",
          text: this.newMessageText.trim(),
        });
        this.newMessageText = "";
        this.$nextTick(() => {
          const messagesContainer = this.$el.querySelector(".messages");
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
      this.newTicketTitle = "";
      this.newTicketDescription = "";
      this.newTicketAssignee = "";
      this.newTicketPriority = "Low";
      this.newTicketDueDate = "";
      this.newTicketStatus = "";
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

.ticket-enter-active,
.ticket-leave-active {
  transition: all 0.5s ease;
}
.ticket-enter,
.ticket-leave-to {
  opacity: 0;
  transform: translateY(20px);
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
  justify-content: space-between;
  gap: 20px;
}
.ticket-column {
  flex: 1;
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
  border-radius: 8px;
  box-shadow: 0 2px 5px rgba(0, 0, 0, 0.1);
  transition: transform 0.2s, background-color 0.5s ease;
  cursor: grab;
}
.ticket-card:hover {
  transform: scale(1.02);
}
.ticket-card:active {
  cursor: grabbing;
}
.ticket-card h3 {
  margin: 0;
  font-weight: 500;
  color: var(--text-color);
}
.ticket-card p {
  margin: 5px 0;
  color: var(--text-color);
}
.ticket-actions {
  margin-top: 10px;
  display: flex;
  gap: 10px;
}
.ticket-actions button {
  flex: 1;
  padding: 5px 10px;
  border: none;
  border-radius: 4px;
  cursor: pointer;
  transition: background-color 0.3s;
}
.btn-edit {
  background-color: #007bff;
  color: white;
}
.btn-edit:hover {
  background-color: #0056b3;
}
.btn-delete {
  background-color: #e74c3c;
  color: white;
}
.btn-delete:hover {
  background-color: #c0392b;
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
  padding: 20px;
  border-radius: 12px;
  width: 400px;
  color: var(--text-color);
  position: relative;
  transition: background-color 0.5s ease;
}
.modal-content h2 {
  margin-bottom: 20px;
  color: var(--header-color);
}
.modal-content input,
.modal-content textarea,
.modal-content select {
  width: 100%;
  margin: 10px 0;
  padding: 10px;
  border-radius: 8px;
  border: none;
  background-color: var(--input-background);
  color: var(--input-text-color);
  outline: none;
  transition: background-color 0.5s ease, color 0.5s ease;
}
.modal-content input::placeholder,
.modal-content textarea::placeholder {
  color: #888;
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
  margin: 5px;
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

/* Chat History Styles */
.chat-history {
  position: fixed;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  background-color: rgba(18, 18, 18, 0.8);
  display: flex;
  flex-direction: column;
  align-items: center;
  padding-top: 50px;
  color: var(--text-color);
  z-index: 1000;
}
.chat-history h2 {
  margin-bottom: 20px;
  color: var(--header-color);
}
.messages {
  width: 80%;
  max-height: 50%;
  overflow-y: auto;
  background-color: var(--ticket-column-background);
  padding: 20px;
  border-radius: 12px;
  margin-bottom: 20px;
  transition: background-color 0.5s ease;
}
.message {
  display: flex;
  flex-direction: column;
  margin-bottom: 10px;
  max-width: 70%;
}
.message.sent {
  align-self: flex-end;
  align-items: flex-end;
}
.message.received {
  align-self: flex-start;
  align-items: flex-start;
}
.message-content {
  background-color: #00d1b2;
  color: #ffffff;
  padding: 10px 15px;
  border-radius: 15px;
  position: relative;
  animation: fadeIn 0.3s ease-in-out;
}
.message.sent .message-content {
  background-color: #007bff;
}
.message-sender {
  font-size: 0.8rem;
  margin-top: 5px;
  color: var(--text-color);
}
.new-message {
  display: flex;
  width: 80%;
  margin-bottom: 20px;
}
.new-message input {
  flex: 1;
  padding: 10px;
  border-radius: 8px 0 0 8px;
  border: none;
  background-color: var(--input-background);
  color: var(--input-text-color);
  transition: background-color 0.5s ease, color 0.5s ease;
}
.new-message button {
  padding: 10px 20px;
  border: none;
  background-color: var(--header-color);
  color: #ffffff;
  border-radius: 0 8px 8px 0;
  cursor: pointer;
}
.new-message button:hover {
  background-color: #00bfa5;
}
.btn-close-chat {
  padding: 10px 20px;
  background-color: #e74c3c;
  color: white;
  border: none;
  border-radius: 8px;
  cursor: pointer;
}
.btn-close-chat:hover {
  background-color: #c0392b;
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
