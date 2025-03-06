<template>
  <div class="min-h-screen bg-gradient-to-br from-blue-50 to-gray-100 flex flex-col items-center p-6">
    <div class="w-full max-w-5xl bg-white shadow-lg rounded-xl p-8 transition-transform transform hover:scale-[1.01]">
      <h1 class="text-4xl font-extrabold text-center mb-8 text-gray-900">Cihazlar Arası Komut Karşılaştırma</h1>

      <!-- Arama ve Filtreleme Alanı -->
      <div class="flex flex-wrap justify-between items-center mb-6 gap-4">
        <input v-model="searchQuery" placeholder="Komut ara..." 
          class="w-full md:w-1/2 p-3 border rounded-xl shadow-sm focus:ring-2 focus:ring-blue-500 outline-none transition" />
        <select v-model="selectedVendor" class="p-3 border rounded-xl shadow-sm focus:ring-2 focus:ring-blue-500 outline-none transition">
          <option value="">Tüm Cihazlar</option>
          <option v-for="vendor in vendors" :key="vendor" :value="vendor">{{ vendor }}</option>
        </select>
      </div>

      <!-- Komut Listesi Tablosu -->
      <div class="overflow-hidden rounded-xl shadow-md border border-gray-200">
        <table class="w-full border-collapse">
          <thead>
            <tr class="bg-blue-400 text-white">
              <th class="p-4 text-left border-b-2 border-gray-300">Komut Adı</th>
              <th class="p-4 text-left border-b-2 border-gray-300">Açıklama</th>
              <th class="p-4 text-left border-b-2 border-gray-300">Cihaz</th>
              <th class="p-4 text-left border-b-2 border-gray-300">İşlemler</th>
            </tr>
          </thead>
          <tbody>
            <tr v-for="command in filteredCommands" :key="command._id" class="bg-white border-b hover:bg-blue-50 transition">
              <td class="p-4">{{ command.name }}</td>
              <td class="p-4">{{ command.description }}</td>
              <td class="p-4">{{ command.vendors.map(v => v.vendor).join(", ") }}</td>
              <td class="p-4 flex space-x-2">
                <button @click="editCommand(command)" class="px-4 py-2 bg-blue-500 text-white rounded-lg hover:bg-blue-600 transition">Düzenle</button>
                <button @click="deleteCommand(command._id)" class="px-4 py-2 bg-red-500 text-white rounded-lg hover:bg-red-600 transition">Sil</button>
              </td>
            </tr>
          </tbody>
        </table>
      </div>

      <!-- Yeni Komut Ekle Butonu -->
      <div class="mt-6 flex justify-end">
        <button @click="openModal" class="px-6 py-3 bg-green-500 text-white rounded-lg hover:bg-green-600 transition">
          Yeni Komut Ekle
        </button>
      </div>
    </div>
  </div>

  <!-- Modal Popup -->
<div v-if="isModalOpen" class="fixed inset-0 bg-gradient-to-br from-gray-200 via-blue-100 to-white bg-opacity-50 flex justify-center items-center z-50">
    <div class="bg-white p-6 rounded-lg shadow-lg w-full md:w-1/2">
      <h2 class="text-2xl font-bold mb-4 text-gray-900">{{ isEditing ? "Komut Düzenle" : "Yeni Komut Ekle" }}</h2>
      <input v-model="newCommand.name" placeholder="Komut Adı" class="w-full p-3 border rounded-xl mb-4 shadow-sm focus:ring-2 focus:ring-blue-500 outline-none transition" />
      <textarea v-model="newCommand.description" placeholder="Açıklama" class="w-full p-3 border rounded-xl mb-4 shadow-sm focus:ring-2 focus:ring-blue-500 outline-none transition"></textarea>
      <input v-model="newCommand.vendorsString" placeholder="Cihaz Adı (Virgülle Ayır)" class="w-full p-3 border rounded-xl mb-4 shadow-sm focus:ring-2 focus:ring-blue-500 outline-none transition" />
      <div class="flex space-x-2 mt-4">
        <button @click="saveCommand" class="px-5 py-3 bg-green-500 text-white rounded-lg hover:bg-green-600 transition">
          {{ isEditing ? "Güncelle" : "Ekle" }}
        </button>
        <button @click="closeModal" class="px-5 py-3 bg-gray-300 text-gray-800 rounded-lg hover:bg-gray-400 transition">İptal</button>
      </div>
    </div>
  </div>
</template>

<script>
import axios from "axios";

export default {
  data() {
    return {
      commands: [],
      searchQuery: "",
      selectedVendor: "",
      newCommand: { name: "", description: "", vendorsString: "" },
      isEditing: false,
      editingId: null,
      isModalOpen: false
    };
  },
  computed: {
    vendors() {
      return [...new Set(this.commands.flatMap(cmd => cmd.vendors.map(v => v.vendor)))];
    },
    filteredCommands() {
      return this.commands.filter(command => 
        (command.name.toLowerCase().includes(this.searchQuery.toLowerCase()) || 
        command.description.toLowerCase().includes(this.searchQuery.toLowerCase())) &&
        (this.selectedVendor === "" || command.vendors.some(v => v.vendor === this.selectedVendor))
      );
    }
  },
  methods: {
    async fetchCommands() {
      try {
        const response = await axios.get("http://localhost:5000/commands");
        this.commands = response.data;
      } catch (error) {
        console.error("API Hatası:", error);
      }
    },
    async saveCommand() {
      if (!this.newCommand.name || !this.newCommand.description) {
        alert("Lütfen tüm alanları doldurun.");
        return;
      }
      this.newCommand.vendors = this.newCommand.vendorsString.split(",").map(v => ({ vendor: v.trim() }));
      try {
        if (this.isEditing) {
          await axios.put(`http://localhost:5000/commands/${this.editingId}`, this.newCommand);
        } else {
          const response = await axios.post("http://localhost:5000/commands", this.newCommand);
          this.commands.push(response.data.command);
        }
        this.resetForm();
        this.fetchCommands();
        this.closeModal();  // Güncelleme sonrası popup kapanacak
      } catch (error) {
        console.error("Komut kaydetme hatası:", error);
      }
    },
    async deleteCommand(commandId) {
      if (!confirm("Bu komutu silmek istediğinizden emin misiniz?")) return;
      try {
        await axios.delete(`http://localhost:5000/commands/${commandId}`);
        this.commands = this.commands.filter(cmd => cmd._id !== commandId);
      } catch (error) {
        console.error("Komut silme hatası:", error);
      }
    },
    editCommand(command) {
      this.newCommand = { 
        name: command.name, 
        description: command.description, 
        vendorsString: command.vendors.map(v => v.vendor).join(", ") 
      };
      this.isEditing = true;
      this.editingId = command._id;
      this.openModal();
    },
    openModal() {
      this.isModalOpen = true;
    },
    closeModal() {
      this.isModalOpen = false;
      this.resetForm();
    },
    resetForm() {
      this.newCommand = { name: "", description: "", vendorsString: "" };
      this.isEditing = false;
      this.editingId = null;
    }
  },
  mounted() {
    this.fetchCommands();
  }
};
</script>
