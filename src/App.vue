<template>
  <div id="app">
    <header>
      <h1>IT Ticket System</h1>
    </header>
    <div class="ticket-container">
      <div v-for="status in ticketStatuses" :key="status" class="ticket-column" @dragover.prevent @drop="onDrop($event, status)">
        <h2>{{ status }}</h2>
        <div v-for="ticket in filteredTickets(status)" :key="ticket.id" class="ticket-card" draggable="true" @dragstart="onDragStart($event, ticket)">
          <h3>{{ ticket.title }}</h3>
          <p>{{ ticket.description }}</p>
          <p><strong>Assignee:</strong> {{ ticket.assignee }}</p>
          <div class="ticket-actions">
            <button @click="editTicket(ticket)" class="btn-edit">Edit</button>
            <button @click="deleteTicket(ticket.id)" class="btn-delete">Delete</button>
          </div>
        </div>
      </div>
    </div>
    <div class="add-ticket-form">
      <h2>Add New Ticket</h2>
      <input v-model="newTicketTitle" placeholder="Ticket Title" />
      <textarea v-model="newTicketDescription" placeholder="Ticket Description"></textarea>
      <input v-model="newTicketAssignee" placeholder="Assignee" />
      <button @click="addTicket" class="btn-add">Add Ticket</button>
    </div>
    <div v-if="isEditing" class="edit-ticket-form">
      <h2>Edit Ticket</h2>
      <input v-model="editTicketTitle" placeholder="Ticket Title" />
      <textarea v-model="editTicketDescription" placeholder="Ticket Description"></textarea>
      <input v-model="editTicketAssignee" placeholder="Assignee" />
      <button @click="saveTicket" class="btn-save">Save Changes</button>
      <button @click="cancelEdit" class="btn-cancel">Cancel</button>
    </div>
  </div>
</template>

<script>
export default {
  name: "App",
  data() {
    return {
      tickets: [
        { id: 1, title: "Bug in Login Page", description: "The login page throws an error.", status: "To Do", assignee: "John Doe" },
        { id: 2, title: "Update Documentation", description: "Add documentation for the new API endpoints.", status: "In Progress", assignee: "Jane Smith" },
      ],
      newTicketTitle: "",
      newTicketDescription: "",
      newTicketAssignee: "",
      editTicketId: null,
      editTicketTitle: "",
      editTicketDescription: "",
      editTicketAssignee: "",
      isEditing: false,
      ticketStatuses: ["To Do", "In Progress", "Done"],
      draggedTicket: null,
    };
  },
  methods: {
    filteredTickets(status) {
      return this.tickets.filter((ticket) => ticket.status === status);
    },
    addTicket() {
      if (this.newTicketTitle && this.newTicketDescription && this.newTicketAssignee) {
        this.tickets.push({
          id: this.tickets.length + 1,
          title: this.newTicketTitle,
          description: this.newTicketDescription,
          status: "To Do",
          assignee: this.newTicketAssignee,
        });
        this.newTicketTitle = "";
        this.newTicketDescription = "";
        this.newTicketAssignee = "";
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
    },
    saveTicket() {
      const ticket = this.tickets.find((ticket) => ticket.id === this.editTicketId);
      if (ticket) {
        ticket.title = this.editTicketTitle;
        ticket.description = this.editTicketDescription;
        ticket.assignee = this.editTicketAssignee;
      }
      this.cancelEdit();
    },
    cancelEdit() {
      this.isEditing = false;
      this.editTicketId = null;
      this.editTicketTitle = "";
      this.editTicketDescription = "";
      this.editTicketAssignee = "";
    },
    moveTicket(ticket, currentStatus) {
      const currentIndex = this.ticketStatuses.indexOf(currentStatus);
      if (currentIndex < this.ticketStatuses.length - 1) {
        ticket.status = this.ticketStatuses[currentIndex + 1];
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
  },
};
</script>

<style scoped>
@import url('https://fonts.googleapis.com/css2?family=Roboto:wght@400;500;700&display=swap');

#app {
  font-family: 'Roboto', sans-serif;
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
.add-ticket-form, .edit-ticket-form {
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
.edit-ticket-form textarea {
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
.btn-add, .btn-save, .btn-cancel {
  padding: 10px 20px;
  background-color: #00d1b2;
  color: white;
  border: none;
  border-radius: 8px;
  cursor: pointer;
  transition: background-color 0.3s;
}
.btn-add:hover, .btn-save:hover, .btn-cancel:hover {
  background-color: #00bfa5;
}
.btn-cancel {
  background-color: #f39c12;
}
.btn-cancel:hover {
  background-color: #e67e22;
}
</style>
