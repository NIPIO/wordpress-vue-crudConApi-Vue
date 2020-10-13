<template>
	<div class="text-center">
		<div>
		    <v-progress-circular v-if="loading" :rotate="360" :size="100" :width="15" :value="value" color="teal">
		      {{ value }}
		    </v-progress-circular>
		</div>
		<v-data-table v-if="!loading" :headers="titulosTabla" :items="entradas" sort-by="id" class="elevation-1" >
	    <template v-slot:top>
	      <v-toolbar flat color="white">
	        <v-toolbar-title>My CRUD</v-toolbar-title>
	        <v-divider class="mx-4" inset vertical ></v-divider>
	        <v-spacer></v-spacer>
	        <v-dialog v-model="dialog" max-width="500px">
	          <template v-slot:activator="{ on, attrs }">
	            <v-btn color="primary" dark class="mb-2" v-bind="attrs" v-on="on" >New Item</v-btn>
	          </template>
	          <v-card>
	            <v-card-title>
	              <span class="headline">{{ formTitle }}</span>
	            </v-card-title>

	            <v-card-text>
	              <v-container>
	                <v-row>
	                  <v-col cols="12" sm="6" md="4">
	                    <v-text-field type="text" v-model="editedItem.title" label="Nombre"></v-text-field>
	                  </v-col>
	                  <v-col cols="12" sm="6" md="4">
	                    <v-text-field type="text" v-model="editedItem.content" label="Contenido"></v-text-field>
	                  </v-col>
	                </v-row>
	              </v-container>
	            </v-card-text>

	            <v-card-actions>
	              <v-spacer></v-spacer>
	              <v-btn color="blue darken-1" text @click="close">Cancelar</v-btn>
	              <v-btn color="blue darken-1" text @click="save">Guardar</v-btn>
	            </v-card-actions>
	          </v-card>
	        </v-dialog>
	      </v-toolbar>
	    </template>
	    <template v-slot:item.actions="{ item }">
	      <v-icon small class="mr-2" @click="editItem(item)" >
	        mdi-pencil
	      </v-icon>
	      <v-icon small @click="deleteItem(item)" >
	        mdi-delete
	      </v-icon>
	    </template>
	    <template v-slot:no-data>
	      <v-btn color="primary" @click="initialize">Reset</v-btn>
	    </template>
	</v-data-table>

	<v-snackbar v-model="snackbar" :color="colorSnackbar" :timeout="2000">
	{{ textoSnackbar }}
	</v-snackbar>
	</div>

</template>

<script>
  export default {
    data: () => ({
		config: {
			headers:{
				Authorization: 'Bearer eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJpc3MiOiJodHRwOlwvXC92dWVqcy50ZXN0XC93b3JkcHJlc3MiLCJpYXQiOjE1OTg2ODAwMzMsIm5iZiI6MTU5ODY4MDAzMywiZXhwIjoxNTk5Mjg0ODMzLCJkYXRhIjp7InVzZXIiOnsiaWQiOjF9fX0.OtyKB1Oq8o3rbaqa7T5h3Rsiet78MXyYyYiWv8YRgUQ' 
			}
		},
		dialog: false,
		titulosTabla: [
			{ text: 'Id', value: 'id' },
			{
			text: 'Titulo',
			align: 'start',
			sortable: false,
			value: 'title',
			},
			{ text: 'Contenido', value: 'content' },
			{ text: 'Fecha', value: 'date' },
			{ text: 'Estado', value: 'status' },
			{ text: 'Acciones', value: 'actions', sortable: false },
		],
		entradas: [],
		editedIndex: -1,
		editedItem: {
			title: '',
			content: '',
		},
		defaultItem: {
			title: '',
			id: 0,
			content: 0,
			date: 0,
			status: 0,
		},
		interval: {},
	    value: 0,
	    loading: true,
	    snackbar: false,
	    colorSnackbar: '',
	    textoSnackbar: '',
    }),
    computed: {
      formTitle () {
        return this.editedIndex === -1 ? 'New Item' : 'Edit Item'
      },
    },

    mounted () {
		this.interval = setInterval(() => {
			if (this.value === 100) {
				this.loading = false;
			}
			this.value += 10
		}, 200)
    },
    
    watch: {
      dialog (val) {
        val || this.close()
      },
    },

    created () {
      this.initialize()
    },

    methods: {
      async initialize () {
          try {
            const infoBackend = await this.axios.get('wp/v2/posts');
            await infoBackend.data.forEach(element => {
              let item = {};
              item.id = element.id;
              item.title = element.title.rendered;
              item.content = this.limpiar(element.content.rendered);
              item.date = element.date;
              item.status = element.status
              this.entradas.push(item)
            })
          } catch (error){
            console.log(error)
          }
      },

      limpiar(value){
        return value.replace(/<\/?[^>]+(>|$)/g, "")
      },

      editItem (item) {
        this.editedIndex = this.entradas.indexOf(item)
        console.log(this.item)
        this.editedItem = Object.assign({}, item)
        console.log(this.editedItem)
        
        this.dialog = true
      },

      async deleteItem (item) {
        const index = this.entradas.indexOf(item)
        const respuestaDelete = confirm('Are you sure you want to delete this item?') 
        if (respuestaDelete) {
        	const deleteBackend = await this.axios.delete(`wp/v2/posts/${item.id}`, this.config);
       		this.entradas.splice(index, 1)
        	this.mostrarSnackbar(true, 'red', 'eliminada')
        } 
      },
      mostrarSnackbar(bool, color, string) {
          this.snackbar  = bool 
          this.colorSnackbar = color 
          this.textoSnackbar = 'Columna ' + string 
      },
      close () {
        this.dialog = false
        this.$nextTick(() => {
          this.editedItem = Object.assign({}, this.defaultItem)
          this.editedIndex = -1
        })
      },

      async save () {
        if (this.editedIndex > -1) {
          //Editar
          try{
          	const putBackend = await this.axios.post(`wp/v2/posts/${this.editedItem.id}`, this.editedItem, this.config);
          	this.mostrarSnackbar(true, 'green', 'editada')
          	Object.assign(this.entradas[this.editedIndex], this.editedItem)
          } catch(error) {
          	this.mostrarSnackbar(true, 'black', error)
          }
        } else {
          //Crear
          const entrada = {
            title: this.editedItem.title,
            content: this.editedItem.content,
          }
          const postBackend = await this.axios.post('wp/v2/posts', entrada, this.config);
          console.log(postBackend)
          //lleno datos que no tengo en el front pero sí en el back cuando genero uno nuevo (ej el id)
          this.editedItem.id = postBackend.data.id
          this.editedItem.date = postBackend.data.date
          this.editedItem.status = postBackend.data.status
          this.entradas.push(this.editedItem)
          this.mostrarSnackbar(true, 'green', 'agregada')
        }
        this.close()
      },
    },
  }
</script>

<style scoped>
.v-progress-circular {
  margin: 1rem;
}
</style>