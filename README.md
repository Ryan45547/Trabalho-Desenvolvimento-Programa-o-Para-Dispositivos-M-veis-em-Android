dependencies {
    implementation("androidx.lifecycle:lifecycle-viewmodel-ktx:2.6.1")
    implementation("androidx.lifecycle:lifecycle-livedata-ktx:2.6.1")
    implementation("androidx.room:room-runtime:2.5.0")
    implementation("androidx.room:room-ktx:2.5.0")
    kapt("androidx.room:room-compiler:2.5.0")
    package com.example.pedidomanager.data

import androidx.room.Entity
import androidx.room.PrimaryKey

@Entity(tableName = "pedidos")
data class Pedido(
    @PrimaryKey(autoGenerate = true) val id: Int = 0,
    val cliente: String,
    val pedido: String,
    val valor: Double,
    val dataSeparacao: String
)
package com.example.pedidomanager.data

import androidx.lifecycle.LiveData
import androidx.room.*

@Dao
interface PedidoDao {
    @Insert
    suspend fun inserir(pedido: Pedido)

    @Delete
    suspend fun deletar(pedido: Pedido)

    @Query("SELECT * FROM pedidos ORDER BY dataSeparacao ASC")
    fun listarTodos(): LiveData<List<Pedido>>
}
🔹 Passo 6: Criar o Banco de Dados (PedidoDatabase.kt)
kotlin
Copiar
Editar
package com.example.pedidomanager.data

import android.content.Context
import androidx.room.Database
import androidx.room.Room
import androidx.room.RoomDatabase

@Database(entities = [Pedido::class], version = 1)
abstract class PedidoDatabase : RoomDatabase() {
    abstract fun pedidoDao(): PedidoDao

    companion object {
        @Volatile
        private var INSTANCE: PedidoDatabase? = null

        fun getDatabase(context: Context): PedidoDatabase {
            return INSTANCE ?: synchronized(this) {
                val instance = Room.databaseBuilder(
                    context.applicationContext,
                    PedidoDatabase::class.java,
                    "pedido_db"
                ).build()
                INSTANCE = instance
                instance
            }
        }
    }
}

package com.example.pedidomanager.repository

import androidx.lifecycle.LiveData
import com.example.pedidomanager.data.Pedido
import com.example.pedidomanager.data.PedidoDao

class PedidoRepository(private val pedidoDao: PedidoDao) {
    val pedidos: LiveData<List<Pedido>> = pedidoDao.listarTodos()

    suspend fun inserir(pedido: Pedido) {
        pedidoDao.inserir(pedido)
    }

    suspend fun deletar(pedido: Pedido) {
        pedidoDao.deletar(pedido)
    }
}package com.example.pedidomanager.viewmodel

import android.app.Application
import androidx.lifecycle.*
import com.example.pedidomanager.data.Pedido
import com.example.pedidomanager.data.PedidoDatabase
import com.example.pedidomanager.repository.PedidoRepository
import kotlinx.coroutines.launch

class PedidoViewModel(application: Application) : AndroidViewModel(application) {
    private val repository: PedidoRepository
    val pedidos: LiveData<List<Pedido>>

    init {
        val pedidoDao = PedidoDatabase.getDatabase(application).pedidoDao()
        repository = PedidoRepository(pedidoDao)
        pedidos = repository.pedidos
    }

    fun inserir(pedido: Pedido) = viewModelScope.launch {
        repository.inserir(pedido)
    }

    fun deletar(pedido: Pedido) = viewModelScope.launch {
        repository.deletar(pedido)
    }
}
package com.example.pedidomanager.ui

import android.view.LayoutInflater
import android.view.ViewGroup
import androidx.recyclerview.widget.RecyclerView
import com.example.pedidomanager.data.Pedido
import com.example.pedidomanager.databinding.ItemPedidoBinding

class PedidoAdapter(private val onDelete: (Pedido) -> Unit) :
    RecyclerView.Adapter<PedidoAdapter.PedidoViewHolder>() {

    private var pedidos: List<Pedido> = emptyList()

    fun setPedidos(lista: List<Pedido>) {
        pedidos = lista
        notifyDataSetChanged()
    }

    inner class PedidoViewHolder(private val binding: ItemPedidoBinding) :
        RecyclerView.ViewHolder(binding.root) {
        fun bind(pedido: Pedido) {
            binding.tvCliente.text = pedido.cliente
            binding.tvPedido.text = pedido.pedido
            binding.tvValor.text = "R$ ${pedido.valor}"
            binding.tvDataSeparacao.text = "Data: ${pedido.dataSeparacao}"

            binding.btnDeletar.setOnClickListener { onDelete(pedido) }
        }
    }

    override fun onCreateViewHolder(parent: ViewGroup, viewType: Int): PedidoViewHolder {
        val binding = ItemPedidoBinding.inflate(LayoutInflater.from(parent.context), parent, false)
        return PedidoViewHolder(binding)
    }

    override fun onBindViewHolder(holder: PedidoViewHolder, position: Int) {
        holder.bind(pedidos[position])
    }

    override fun getItemCount() = pedidos.size
}
package com.example.pedidomanager.ui

import android.os.Bundle
import androidx.activity.viewModels
import androidx.appcompat.app.AppCompatActivity
import androidx.lifecycle.Observer
import com.example.pedidomanager.databinding.ActivityMainBinding
import com.example.pedidomanager.viewmodel.PedidoViewModel

class MainActivity : AppCompatActivity() {
    private lateinit var binding: ActivityMainBinding
    private val pedidoViewModel: PedidoViewModel by viewModels()
    private val adapter = PedidoAdapter { pedidoViewModel.deletar(it) }

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        binding = ActivityMainBinding.inflate(layoutInflater)
        setContentView(binding.root)

        binding.recyclerPedidos.adapter = adapter

        pedidoViewModel.pedidos.observe(this, Observer {
            adapter.setPedidos(it)
        })
    }
}
