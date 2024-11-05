<template>
  <div id="app">
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
      </div>
    </div>
    <div class="add-ticket-form">
      <h2>Add New Ticket</h2>
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
    </div>
    <div v-if="isEditing" class="edit-ticket-form">
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
    <div v-if="showChat" class="chat-history">
      <h2>Chat History for {{ selectedTicket.title }}</h2>
      <div class="messages">
        <div
          v-for="(message, index) in selectedTicket.chatHistory"
          :key="index"
          class="message"
        >
          <p><strong>{{ message.sender }}:</strong> {{ message.text }}</p>
        </div>
      </div>
      <div class="new-message">
        <input v-model="newMessageText" placeholder="Type a message..." />
        <button @click="sendMessage">Send</button>
      </div>
      <button @click="closeChat" class="btn-close-chat">Close Chat</button>
    </div>
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
          status: "To Do",
          assignee: this.newTicketAssignee,
          priority: this.newTicketPriority,
          dueDate: this.newTicketDueDate,
          chatHistory: [],
        });
        this.newTicketTitle = "";
        this.newTicketDescription = "";
        this.newTicketAssignee = "";
        this.newTicketPriority = "Low";
        this.newTicketDueDate = "";
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
      }
    },
  },
};
</script>

<style scoped>
@import url("https://fonts.googleapis.com/css2?family=Roboto:wght@400;500;700&display=swap");

#app {
  font-family: "Roboto", sans-serif;
  max-width: 1200px;
  margin: 0 auto;
  padding: 20px;
  color: #ffffff;
  background-color: #121212;
  border-radius: 12px;
  box-shadow: 0 4px 15px rgba(0, 0, 0, 0.4);
}
header {
  text-align: center;
  margin-bottom: 30px;
}
header h1 {
  font-size: 2.5rem;
  font-weight: 700;
  color: #00d1b2;
}

/* Navbar Styles */
.navbar {
  background-color: #1f1f1f;
  padding: 10px 20px;
  margin-bottom: 20px;
  border-radius: 12px;
}
.navbar ul {
  list-style-type: none;
  display: flex;
  gap: 15px;
  margin: 0;
  padding: 0;
}
.navbar ul li a {
  color: #00d1b2;
  text-decoration: none;
  font-weight: 500;
  transition: color 0.3s;
}
.navbar ul li a:hover {
  color: #00bfa5;
}

.ticket-container {
  display: flex;
  justify-content: space-between;
  gap: 20px;
}
.ticket-column {
  flex: 1;
  background: #1f1f1f;
  padding: 20px;
  border-radius: 12px;
  border: 1px solid #2c2c2c;
  box-shadow: 0 4px 10px rgba(0, 0, 0, 0.3);
}
.ticket-column h2 {
  font-size: 1.8rem;
  margin-bottom: 15px;
  color: #00d1b2;
}
.ticket-card {
  background: #282828;
  padding: 15px;
  margin-bottom: 15px;
  border-radius: 8px;
  box-shadow: 0 2px 5px rgba(0, 0, 0, 0.4);
  transition: transform 0.2s;
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
  color: #ffffff;
}
.ticket-card p {
  margin: 5px 0;
  color: #bbbbbb;
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
.add-ticket-form,
.edit-ticket-form {
  margin-top: 30px;
  text-align: center;
  background-color: #1f1f1f;
  padding: 20px;
  border-radius: 12px;
  box-shadow: 0 4px 10px rgba(0, 0, 0, 0.3);
}
.add-ticket-form input,
.add-ticket-form textarea,
.edit-ticket-form input,
.edit-ticket-form textarea,
.add-ticket-form select,
.edit-ticket-form select {
  width: 90%;
  margin: 10px 0;
  padding: 10px;
  border-radius: 8px;
  border: none;
  background-color: #333333;
  color: #ffffff;
  outline: none;
}
.add-ticket-form input::placeholder,
.add-ticket-form textarea::placeholder,
.edit-ticket-form input::placeholder,
.edit-ticket-form textarea::placeholder {
  color: #888;
}
.btn-add,
.btn-save,
.btn-cancel {
  padding: 10px 20px;
  background-color: #00d1b2;
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

/* Chat History Styles */
.chat-history {
  position: fixed;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  background-color: rgba(18, 18, 18, 0.95);
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  color: #ffffff;
  z-index: 1000;
}
.chat-history h2 {
  margin-bottom: 20px;
  color: #00d1b2;
}
.messages {
  width: 80%;
  max-height: 50%;
  overflow-y: auto;
  background-color: #1f1f1f;
  padding: 20px;
  border-radius: 12px;
  margin-bottom: 20px;
}
.message {
  margin-bottom: 10px;
}
.message p {
  margin: 0;
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
  background-color: #333333;
  color: #ffffff;
}
.new-message button {
  padding: 10px 20px;
  border: none;
  background-color: #00d1b2;
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
</style>
