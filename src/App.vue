<template>
  <div id="app">
    <header>
      <h1>IT Ticket System</h1>
    </header>
    <div class="ticket-container">
      <div v-for="status in ticketStatuses" :key="status" class="ticket-column">
        <h2>{{ status }}</h2>
        <div v-for="ticket in filteredTickets(status)" :key="ticket.id" class="ticket-card">
          <h3>{{ ticket.title }}</h3>
          <p>{{ ticket.description }}</p>
          <p><strong>Assignee:</strong> {{ ticket.assignee }}</p>
          <button @click="editTicket(ticket)">Edit</button>
          <button @click="deleteTicket(ticket.id)">Delete</button>
          <button @click="moveTicket(ticket, status)">Move</button>
        </div>
      </div>
    </div>
    <div class="add-ticket-form">
      <h2>Add New Ticket</h2>
      <input v-model="newTicketTitle" placeholder="Ticket Title" />
      <textarea v-model="newTicketDescription" placeholder="Ticket Description"></textarea>
      <input v-model="newTicketAssignee" placeholder="Assignee" />
      <button @click="addTicket">Add Ticket</button>
    </div>
    <div v-if="isEditing" class="edit-ticket-form">
      <h2>Edit Ticket</h2>
      <input v-model="editTicketTitle" placeholder="Ticket Title" />
      <textarea v-model="editTicketDescription" placeholder="Ticket Description"></textarea>
      <input v-model="editTicketAssignee" placeholder="Assignee" />
      <button @click="saveTicket">Save Changes</button>
      <button @click="cancelEdit">Cancel</button>
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
  },
};
</script>

<style scoped>
#app {
  font-family: Arial, sans-serif;
  max-width: 1200px;
  margin: 0 auto;
  padding: 20px;
}
header {
  text-align: center;
  margin-bottom: 20px;
}
.ticket-container {
  display: flex;
  justify-content: space-between;
}
.ticket-column {
  flex: 1;
  margin: 0 10px;
  background: #f4f4f4;
  padding: 10px;
  border-radius: 8px;
}
.ticket-card {
  background: #ffffff;
  padding: 10px;
  margin-bottom: 10px;
  border-radius: 8px;
  box-shadow: 0 2px 5px rgba(0, 0, 0, 0.1);
}
.add-ticket-form, .edit-ticket-form {
  margin-top: 20px;
  text-align: center;
}
.add-ticket-form input,
.add-ticket-form textarea,
.edit-ticket-form input,
.edit-ticket-form textarea {
  width: 80%;
  margin: 10px 0;
  padding: 8px;
  border-radius: 4px;
  border: 1px solid #ccc;
}
.add-ticket-form button, .edit-ticket-form button {
  padding: 10px 20px;
  background-color: #007bff;
  color: white;
  border: none;
  border-radius: 4px;
  cursor: pointer;
}
.add-ticket-form button:hover, .edit-ticket-form button:hover {
  background-color: #0056b3;
}
</style>