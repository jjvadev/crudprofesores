<template>
  <div class="shell">
    <!-- TOPBAR -->
    <header class="nav no-print">
      <div class="brand">
        <div class="logo">🎓</div>
        <div class="brand-text">
          <div class="title">Gestor de Profesores</div>
          <div class="subtitle">Vue 3 · LocalStorage</div>
        </div>
      </div>

      <div class="actions">
        <button class="btn primary" @click="openCreate">➕ Nuevo</button>

        <label class="btn" title="Importar JSON">
          ⬆️ Importar
          <input class="file" type="file" accept="application/json" @change="handleImport" />
        </label>

        <button class="btn" @click="exportJSON" title="Exportar JSON">⬇️ Exportar</button>
        <button class="btn" @click="printAll" title="Imprimir lista">🖨️ Imprimir</button>
        <button class="btn danger" @click="resetDB" title="Borrar datos locales">🗑️ Reset</button>

        <button class="btn" @click="toggleSelectionMode">
          {{ selectionMode ? 'Cancelar selección' : 'Seleccionar' }}
        </button>
      </div>
    </header>

    <!-- FILTERS -->
    <section class="filters no-print">
      <div class="field wide">
        <label>Cédula (Enter abre la ficha)</label>
        <div class="row">
          <input v-model.trim="quickCedula" @keyup.enter="goQuickCedula" placeholder="Ingresa cédula y presiona Enter" />
          <button class="btn" @click="goQuickCedula">Ir</button>
        </div>
      </div>
      <div class="field">
        <label>Buscar por cédula o nombre</label>
        <input v-model.trim="query" placeholder="Ej: 1012345678 o Ana" />
      </div>
      <div class="field">
        <label>Tipo</label>
        <select v-model="filtroTipo">
          <option value="">Todos</option>
          <option value="Planta">Planta</option>
          <option value="Catedrático">Catedrático</option>
          <option value="Contrato">Contrato</option>
        </select>
      </div>
      <div class="field compact">
        <label>Vista</label>
        <div class="segmented">
          <button :class="{on: view==='cards'}" @click="view='cards'">Tarjetas</button>
          <button :class="{on: view==='table'}" @click="view='table'">Tabla</button>
          <button :class="{on: view==='semana'}" @click="view='semana'">Semana</button>
        </div>
      </div>
      <div class="counter">
        <span class="chip">{{ filtrados.length }} resultado(s)</span>
      </div>
    </section>

    <!-- BULK ACTIONS -->
    <section class="bulkbar no-print" v-if="selectionMode">
      <div class="bulk-left"><strong>{{ selectedCount }}</strong> seleccionado(s)</div>
      <div class="bulk-right">
        <button class="btn" @click="selectAllCurrent">Seleccionar visibles</button>
        <button class="btn" @click="clearSelection">Limpiar</button>
        <button class="btn" @click="printSelected">Imprimir selección</button>
        <button class="btn primary" @click="exportSelectedPDF">Exportar PDF</button>
        <button class="btn" @click="toggleSelectionMode">Salir</button>
      </div>
    </section>

    <!-- CARDS -->
    <section v-if="view==='cards'" class="cards">
      <div v-if="filtrados.length===0" class="empty">Sin resultados. Agrega profesores o ajusta los filtros.</div>

      <article v-for="p in filtrados" :key="p.cedula" class="card">
        <i class="card-bar" aria-hidden></i>

        <label v-if="selectionMode" class="select-box">
          <input type="checkbox" :checked="isSelected(p)" @change="toggleSelect(p)" />
        </label>

        <div class="card-head">
          <div class="person">
            <div class="avatar" aria-hidden>👤</div>
            <div>
              <div class="name">{{ p.nombre }}</div>
              <div class="id mono">{{ p.cedula }}</div>
            </div>
          </div>
          <span :class="['pill', typePill(p.tipo)]">{{ p.tipo }}</span>
        </div>

        <div class="grid">
          <div>
            <div class="muted">Horas semanales</div>
            <div>{{ p.horasSemanales ?? '—' }}</div>
          </div>
          <div>
            <div class="muted">Correo</div>
            <div>{{ p.email || '—' }}</div>
          </div>
          <div>
            <div class="muted">Ubicación de trabajo</div>
            <div>{{ p.ubicacionTrabajo || '—' }}</div>
          </div>
          <div>
            <div class="muted">Residencia</div>
            <div>{{ p.lugarResidencia || '—' }}</div>
          </div>
        </div>

        <div v-if="(p.actividadesLista && p.actividadesLista.length) || p.actividades" class="activities">
          <template v-if="p.actividadesLista && p.actividadesLista.length">
            <span v-for="(a,i) in p.actividadesLista" :key="i" class="act-chip" :title="a.nota || ''">
              {{ a.nombre }} <small v-if="a.horas">({{ a.horas }}h)</small>
            </span>
          </template>
          <template v-else>
            <span class="act-note">{{ p.actividades }}</span>
          </template>
        </div>

        <div class="card-actions">
          <button class="link" @click="verDetalle(p)">Ver</button>
          <button class="link" @click="openEdit(p)">Editar</button>
          <button class="link" @click="imprimirUno(p)">Imprimir</button>
          <button class="link danger" @click="removeTeacher(p)">Eliminar</button>
        </div>
      </article>
    </section>

    <!-- TABLE -->
    <section v-else-if="view==='table'" class="table-card">
      <div class="table-wrapper">
        <table class="table">
          <thead>
            <tr>
              <th v-if="selectionMode" style="width:44px">
                <input type="checkbox" :checked="allVisibleSelected" @change="toggleSelectAllVisible" />
              </th>
              <th>Cédula</th>
              <th>Nombre</th>
              <th>Tipo</th>
              <th>Horas/Sem</th>
              <th>Ubicación</th>
              <th>Residencia</th>
              <th class="text-right">Acciones</th>
            </tr>
          </thead>
          <tbody>
            <tr v-if="filtrados.length === 0">
              <td class="empty" :colspan="selectionMode?8:7">Sin resultados. Agrega profesores o ajusta los filtros.</td>
            </tr>
            <tr v-for="p in filtrados" :key="p.cedula">
              <td v-if="selectionMode">
                <input type="checkbox" :checked="isSelected(p)" @change="toggleSelect(p)" />
              </td>
              <td class="mono">{{ p.cedula }}</td>
              <td class="semi">{{ p.nombre }}</td>
              <td><span :class="['pill', typePill(p.tipo)]">{{ p.tipo }}</span></td>
              <td>{{ p.horasSemanales ?? '—' }}</td>
              <td>{{ p.ubicacionTrabajo || '—' }}</td>
              <td>{{ p.lugarResidencia || '—' }}</td>
              <td>
                <div class="row-actions">
                  <button class="link" @click="verDetalle(p)">Ver</button>
                  <button class="link" @click="openEdit(p)">Editar</button>
                  <button class="link" @click="imprimirUno(p)">Imprimir</button>
                  <button class="link danger" @click="removeTeacher(p)">Eliminar</button>
                </div>
              </td>
            </tr>
          </tbody>
        </table>
      </div>
    </section>

    <!-- WEEKLY PLANNER -->
    <section v-else class="planner">
      <div class="planner-head">
        <div class="legend">
          <span class="legend-item planta">Planta</span>
          <span class="legend-item catedra">Catedrático</span>
          <span class="legend-item contrato">Contrato</span>
          <span class="legend-item mode">Virtual (borde punteado)</span>
        </div>
        <div class="muted">Vista semanal de horarios ({{ startHour }}:00–{{ endHour }}:00)</div>
      </div>

      <div class="planner-grid">
        <div class="hours-rail">
          <div v-for="h in hourMarks" :key="h" class="hour">{{ pad(h) }}:00</div>
        </div>
        <div class="days" :style="{height: plannerHeight+'px'}">
          <div class="day-header" v-for="d in days" :key="d">{{ d }}</div>

          <div class="day-col" v-for="(d,di) in days" :key="'col-'+d" :style="{height: plannerHeight+'px'}">
            <div v-for="h in hourMarks" :key="d+'-'+h" class="hour-line"></div>

            <div
              v-for="(b, i) in semanaBlocks.filter(x=>x.dayIndex===di)"
              :key="d+'-b-'+i"
              class="block"
              :class="[b.kind, b.mode]"
              :style="{ top: b.top+'px', height: b.height+'px' }"
              :title="b.title+' — '+b.subtitle"
            >
              <div class="b-title">{{ b.title }}</div>
              <div class="b-sub">{{ b.subtitle }}</div>
            </div>
          </div>
        </div>
      </div>

      <div v-if="semanaBlocks.length===0" class="empty mt">No hay horarios en la vista actual.</div>

      <div v-if="virtualSinHorario.length" class="virtual-panel mt">
        <div class="virtual-head">Clases virtuales sin horario ({{ virtualSinHorario.length }})</div>
        <ul class="virtual-list">
          <li v-for="(v,i) in virtualSinHorario" :key="'virt-'+i">
            <span class="v-materia">{{ v.materia }}</span>
            <span class="v-sep">•</span>
            <span class="v-prof">{{ v.profesor }}</span>
            <span class="v-sep" v-if="v.programa">•</span>
            <span class="v-prog" v-if="v.programa">{{ v.programa }}</span>
            <span class="v-aula" v-if="v.aula"> ({{ v.aula }})</span>
          </li>
        </ul>
      </div>
    </section>

    <!-- TOASTS -->
    <div class="toasts">
      <div v-for="t in toasts" :key="t.id" class="toast">{{ t.msg }}</div>
    </div>

    <!-- MODAL FORM -->
    <div v-if="showForm" class="overlay">
      <div class="modal">
        <div class="modal-head">
          <div>
            <h2>{{ formMode === 'create' ? 'Nuevo profesor' : 'Editar profesor' }}</h2>
            <small class="muted">Obligatorios: <span class="req">Cédula</span>, <span class="req">Nombre</span> y <span class="req">Tipo</span>.</small>
          </div>
          <button class="icon-btn" @click="closeForm" aria-label="Cerrar">✖</button>
        </div>

        <form class="form-grid" @submit.prevent="save">
          <div class="field">
            <label>Cédula *</label>
            <input v-model.trim="form.cedula" :disabled="formMode==='edit'" required placeholder="1012345678" />
          </div>
          <div class="field">
            <label>Nombre completo *</label>
            <input v-model.trim="form.nombre" required placeholder="Ana María Pérez" />
          </div>
          <div class="field">
            <label>Tipo *</label>
            <select v-model="form.tipo" required>
              <option>Planta</option>
              <option>Catedrático</option>
              <option>Contrato</option>
            </select>
          </div>
          <div class="field">
            <label>Horas semanales</label>
            <input v-model.number="form.horasSemanales" type="number" min="0" placeholder="16" />
          </div>

          <div class="col-2">
            <div class="subhead">
              <label>Actividades (chips) — opcional</label>
              <button type="button" class="btn" @click="addActividad">Añadir actividad</button>
            </div>
            <div v-if="!form.actividadesLista.length" class="muted">Agrega actividades con nombre y horas semanales.</div>
            <div v-for="(a, ai) in form.actividadesLista" :key="'act-'+ai" class="actividad-row">
              <input v-model.trim="a.nombre" placeholder="Nombre (ej. Tutorías)" />
              <input v-model.number="a.horas" min="0" type="number" placeholder="Horas/sem" />
              <input v-model.trim="a.nota" placeholder="Nota (opcional)" />
              <button type="button" class="icon-btn" @click="form.actividadesLista.splice(ai,1)">🗑️</button>
            </div>
          </div>

          <div class="field col-2">
            <label>Actividades (texto libre)</label>
            <textarea v-model.trim="form.actividades" rows="2" placeholder="Clases, investigación, tutorías…"></textarea>
          </div>

          <div class="field">
            <label>Ubicación de trabajo</label>
            <input v-model.trim="form.ubicacionTrabajo" placeholder="Bloque A - Of. 203" />
          </div>
          <div class="field">
            <label>Lugar de residencia</label>
            <input v-model.trim="form.lugarResidencia" placeholder="Barrio/Ciudad" />
          </div>
          <div class="field">
            <label>Teléfono</label>
            <input v-model.trim="form.telefono" placeholder="Opcional" />
          </div>
          <div class="field">
            <label>Correo</label>
            <input v-model.trim="form.email" type="email" placeholder="Opcional" />
          </div>

          <div class="col-2">
            <div class="subhead">
              <label>Horarios (opcional)</label>
              <div class="sub-actions">
                <button type="button" class="btn" @click="addHorario">Añadir horario</button>
                <button type="button" class="btn subtle" @click="addHorarioPlantilla('Lunes','08:00','10:00','Clase','A-101')">+ Plantilla</button>
              </div>
            </div>
            <div v-if="form.horarios.length===0" class="muted">Aún no has agregado horarios.</div>

            <div v-for="(h,idx) in form.horarios" :key="'h-'+idx" class="horario">
              <select v-model="h.dia" :disabled="h.modalidad==='Virtual'">
                <option value="Lunes">Lunes</option>
                <option value="Martes">Martes</option>
                <option value="Miércoles">Miércoles</option>
                <option value="Jueves">Jueves</option>
                <option value="Viernes">Viernes</option>
                <option value="Sábado">Sábado</option>
                <option value="Domingo">Domingo</option>
              </select>
              <input v-model="h.inicio" type="time" :disabled="h.modalidad==='Virtual'" />
              <input v-model="h.fin" type="time" :disabled="h.modalidad==='Virtual'" />
              <select v-model="h.modalidad" title="Modalidad">
                <option value="Presencial">Presencial</option>
                <option value="Virtual">Virtual</option>
              </select>
              <input v-model.trim="h.materia" placeholder="Materia" />
              <input v-model.trim="h.programa" placeholder="Programa/Facultad" />
              <div class="horario-right">
                <input v-model.trim="h.aula" placeholder="Aula" />
                <button type="button" class="icon-btn" @click="form.horarios.splice(idx,1)">🗑️</button>
              </div>
            </div>

            <div v-if="horarioWarnings.length" class="warnings">
              <div v-for="(w,i) in horarioWarnings" :key="'w-'+i">⚠️ {{ w }}</div>
            </div>
          </div>

          <div class="form-actions col-2">
            <button type="button" class="btn" @click="closeForm">Cancelar</button>
            <button type="submit" class="btn primary">Guardar</button>
          </div>
        </form>
      </div>
    </div>

    <!-- MODAL DETAIL -->
    <div v-if="showDetail" class="overlay">
      <div class="modal">
        <div class="modal-head">
          <h2>Ficha de profesor</h2>
          <button class="icon-btn" @click="showDetail=false">✖</button>
        </div>

        <div v-if="detalle" class="detail print-card">
          <div class="detail-top">
            <div>
              <div class="muted">Cédula</div>
              <div class="mono">{{ detalle.cedula }}</div>
            </div>
            <span :class="['pill', typePill(detalle.tipo)]">{{ detalle.tipo }}</span>
          </div>

          <h3 class="detail-name">{{ detalle.nombre }}</h3>

          <div class="detail-grid">
            <div>
              <div class="muted">Horas semanales</div>
              <div>{{ detalle.horasSemanales ?? '—' }}</div>
            </div>
            <div>
              <div class="muted">Correo</div>
              <div>{{ detalle.email || '—' }}</div>
            </div>
            <div>
              <div class="muted">Ubicación de trabajo</div>
              <div>{{ detalle.ubicacionTrabajo || '—' }}</div>
            </div>
            <div>
              <div class="muted">Lugar de residencia</div>
              <div>{{ detalle.lugarResidencia || '—' }}</div>
            </div>

            <div class="col-2" v-if="detalle.actividadesLista && detalle.actividadesLista.length">
              <div class="muted">Actividades</div>
              <div class="act-chips">
                <span v-for="(a,i) in detalle.actividadesLista" :key="'da-'+i" class="act-chip" :title="a.nota || ''">
                  {{ a.nombre }} <small v-if="a.horas">({{ a.horas }}h)</small>
                </span>
              </div>
            </div>
            <div class="col-2" v-if="detalle.actividades && !detalle.actividadesLista?.length">
              <div class="muted">Actividades (nota)</div>
              <div class="pre">{{ detalle.actividades }}</div>
            </div>
          </div>

          <div class="mt">
            <div class="muted mb">Horarios</div>
            <div v-if="!detalle.horarios || detalle.horarios.length===0" class="muted">—</div>
            <div v-else class="table-wrapper">
              <table class="table compact">
                <thead>
                  <tr>
                    <th>Día</th>
                    <th>Inicio</th>
                    <th>Fin</th>
                    <th>Materia</th>
                    <th>Programa</th>
                    <th>Aula</th>
                    <th>Modalidad</th>
                  </tr>
                </thead>
                <tbody>
                  <tr v-for="(h,i) in detalle.horarios" :key="'dh-'+i">
                    <td>{{ h.modalidad==='Virtual' && (!h.dia||!h.inicio||!h.fin) ? '—' : (h.dia || '—') }}</td>
                    <td>{{ h.modalidad==='Virtual' && (!h.dia||!h.inicio||!h.fin) ? '—' : (h.inicio || '—') }}</td>
                    <td>{{ h.modalidad==='Virtual' && (!h.dia||!h.inicio||!h.fin) ? '—' : (h.fin || '—') }}</td>
                    <td>{{ h.materia || '—' }}</td>
                    <td>{{ h.programa || '—' }}</td>
                    <td>{{ h.aula || '—' }}</td>
                    <td>{{ h.modalidad || '—' }}</td>
                  </tr>
                </tbody>
              </table>
            </div>
          </div>

          <div class="detail-actions no-print">
            <button class="btn" @click="imprimirUno(detalle)">Imprimir</button>
            <button class="btn" @click="showDetail=false">Cerrar</button>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, computed, watch, onMounted } from 'vue'

/* ===== Constantes ===== */
const STORAGE_KEY = 'db_profesores_v1'
const startHour = 6
const endHour = 22
const pxPerMin = 1
const plannerHeight = (endHour - startHour) * 60 * pxPerMin
const days = ['Lunes','Martes','Miércoles','Jueves','Viernes','Sábado','Domingo']
const hourMarks = Array.from({length: endHour - startHour + 1}, (_,i)=> i + startHour)

/* ===== Estado principal ===== */
const profesores = ref([])
const query = ref('')
const filtroTipo = ref('')
const quickCedula = ref('')
const view = ref('cards') // 'cards' | 'table' | 'semana'

/* Modales y formularios */
const showForm = ref(false)
const showDetail = ref(false)
const detalle = ref(null)
const formMode = ref('create') // 'create' | 'edit'
const form = ref(emptyForm())
const toasts = ref([])
const horarioWarnings = ref([])

/* Selección para imprimir/exportar */
const selectionMode = ref(false)
const selected = ref(new Set())

/* ===== Computados ===== */
const filtrados = computed(() => {
  const q = query.value.trim().toLowerCase()
  return profesores.value
    .filter(p => !filtroTipo.value || p.tipo === filtroTipo.value)
    .filter(p => !q ? true : String(p.cedula).toLowerCase().includes(q) || (p.nombre || '').toLowerCase().includes(q))
    .sort((a, b) => (a.nombre || '').localeCompare(b.nombre || ''))
})

const selectedCount = computed(() => selected.value.size)
const allVisibleSelected = computed(() =>
  filtrados.value.length > 0 &&
  filtrados.value.every(p => selected.value.has(String(p.cedula).trim()))
)

const semanaBlocks = computed(() => {
  // Bloques sólo para horarios con día y horas válidos
  const blocks = []
  for (const p of filtrados.value) {
    if (!Array.isArray(p.horarios)) continue
    for (const h of p.horarios) {
      // si es virtual sin horario, no aparece en la grilla semanal
      const isVirtual = h.modalidad === 'Virtual'
      const hasSchedule = !h.sinHorario && h.dia && h.inicio && h.fin
      if (!hasSchedule) continue

      const di = dayIndex(h.dia)
      const s = toMinutes(h.inicio)
      const e = toMinutes(h.fin)
      if (di < 0 || s == null || e == null || e <= s) continue

      const mode = isVirtual ? 'm-virtual' : 'm-presencial'
      const emoji = isVirtual ? '💻 Virtual' : '🏫 Presencial'
      const prog = h.programa ? ` • ${h.programa}` : ''
      blocks.push({
        dayIndex: di,
        top: (s - startHour * 60) * pxPerMin,
        height: (e - s) * pxPerMin,
        kind: kindClass(p.tipo),
        mode,
        title: (h.materia || 'Clase') + (h.aula ? ` · ${h.aula}` : ''),
        subtitle: `${emoji}${prog} • ${p.nombre} (${p.tipo})`,
      })
    }
  }
  return blocks
})

const virtualSinHorario = computed(() => {
  // Lista de clases virtuales sin día/hora para mostrarlas aparte
  const out = []
  for (const p of filtrados.value) {
    for (const h of (p.horarios || [])) {
      if (h?.modalidad === 'Virtual' && (h.sinHorario || (!h.dia || !h.inicio || !h.fin))) {
        out.push({
          profesor: p.nombre,
          materia: h.materia || 'Clase virtual',
          programa: h.programa || '',
          aula: h.aula || ''
        })
      }
    }
  }
  return out
})

/* ===== Ciclo de vida ===== */
onMounted(() => {
  loadDB()
  if (profesores.value.length === 0) {
    // Datos de ejemplo
    profesores.value = [
      { cedula: '1012345678', nombre: 'Ana María Pérez', tipo: 'Planta', horasSemanales: 16,
        actividadesLista: [{ nombre: 'Tutorías', horas: 2 }, { nombre: 'Investigación', horas: 4 }],
        actividades: 'Clases de Cálculo I y asesorías.',
        ubicacionTrabajo: 'Bloque A - Of. 203', lugarResidencia: 'Laureles, Medellín',
        telefono: '3001234567', email: 'ana.perez@uni.edu',
        horarios: [
          { dia: 'Lunes', inicio: '08:00', fin: '10:00', modalidad: 'Presencial', sinHorario: false, materia: 'Cálculo I', programa: 'Ingeniería', aula: 'A-201' },
          { modalidad: 'Virtual', sinHorario: true, materia: 'Cálculo I (apoyo virtual)', programa: 'Ingeniería' }
        ] },
      { cedula: '1098765432', nombre: 'Carlos Gómez', tipo: 'Catedrático', horasSemanales: 8,
        actividadesLista: [{ nombre: 'Cátedra', horas: 8 }],
        actividades: 'Cátedra de Programación.',
        ubicacionTrabajo: 'Bloque B - Lab. 1', lugarResidencia: 'Envigado',
        telefono: '', email: 'carlos.gomez@uni.edu', horarios: [] },
      { cedula: '1029384756', nombre: 'Luisa Fernández', tipo: 'Contrato', horasSemanales: 12,
        actividadesLista: [{ nombre: 'Proyecto', horas: 12 }],
        actividades: 'Proyecto de investigación.',
        ubicacionTrabajo: 'Bloque C - Of. 110', lugarResidencia: 'Bello',
        telefono: '', email: '', horarios: [] },
    ]
    saveDB()
    toast('Base de ejemplo cargada')
  }
})

/* Validación dinámica de horarios del formulario */
watch(() => form.value.horarios, () => {
  horarioWarnings.value = validateHorarios(form.value.horarios)
}, { deep: true })

/* ===== Utilidades ===== */
function emptyForm () {
  return {
    cedula: '',
    nombre: '',
    tipo: 'Planta',
    horasSemanales: null,
    actividades: '',
    actividadesLista: [],
    ubicacionTrabajo: '',
    lugarResidencia: '',
    telefono: '',
    email: '',
    horarios: [],
  }
}

function normalizeHorario(h){
  const hasSch = h?.inicio && h?.fin
  const isVirtual = h?.modalidad === 'Virtual'
  const sinHorario = h?.sinHorario ?? (isVirtual && !hasSch)
  return {
    dia: h?.dia || '',
    inicio: h?.inicio || '',
    fin: h?.fin || '',
    modalidad: isVirtual ? 'Virtual' : (h?.modalidad || (hasSch ? 'Presencial' : 'Virtual')),
    sinHorario,
    materia: h?.materia || '',
    programa: h?.programa || '',
    aula: h?.aula || ''
  }
}

function toast (msg) {
  const id = Math.random().toString(36).slice(2)
  toasts.value.push({ id, msg })
  setTimeout(() => { toasts.value = toasts.value.filter(t => t.id !== id) }, 2200)
}
function typePill (tipo) {
  switch (tipo) {
    case 'Planta': return 'pill--planta'
    case 'Catedrático': return 'pill--catedra'
    case 'Contrato': return 'pill--contrato'
    default: return 'pill--neutral'
  }
}
function kindClass (tipo) {
  return tipo === 'Planta' ? 'k-planta' : tipo === 'Catedrático' ? 'k-catedra' : 'k-contrato'
}
function pad(n){ return String(n).padStart(2,'0') }
function dayIndex(d) {
  const map = { 'Lunes':0,'Martes':1,'Miércoles':2,'Miercoles':2,'Jueves':3,'Viernes':4,'Sábado':5,'Sabado':5,'Domingo':6 }
  return map[d] ?? -1
}
function toMinutes(t) {
  if (!t || !/^\d{2}:\d{2}$/.test(t)) return null
  const [hh, mm] = t.split(':').map(Number)
  return hh * 60 + mm
}
function mins(m){ return `${pad(Math.floor(m/60))}:${pad(m%60)}` }

/* ===== Base de datos (LocalStorage) ===== */
function loadDB () {
  try {
    const raw = localStorage.getItem(STORAGE_KEY)
    profesores.value = raw ? JSON.parse(raw) : []
  } catch {
    profesores.value = []
  }
}
function saveDB () { localStorage.setItem(STORAGE_KEY, JSON.stringify(profesores.value)) }
function resetDB () {
  if (!confirm('Esto borrará todos los datos locales. ¿Continuar?')) return
  localStorage.removeItem(STORAGE_KEY)
  profesores.value = []
  toast('DB reiniciada')
}

/* ===== CRUD ===== */
function openCreate () {
  formMode.value = 'create'
  form.value = emptyForm()
  horarioWarnings.value = []
  showForm.value = true
}
function openEdit (p) {
  formMode.value = 'edit'
  const clone = JSON.parse(JSON.stringify(p))
  clone.horarios = Array.isArray(clone.horarios) ? clone.horarios.map(normalizeHorario) : []
  if (!Array.isArray(clone.actividadesLista)) clone.actividadesLista = []
  form.value = clone
  horarioWarnings.value = validateHorarios(form.value.horarios || [])
  showForm.value = true
}
function closeForm () {
  showForm.value = false
  form.value = emptyForm()
  horarioWarnings.value = []
}
function addHorario () {
  form.value.horarios.push({
    dia: 'Lunes', inicio: '', fin: '', modalidad: 'Presencial', sinHorario: false, materia: '', programa: '', aula: ''
  })
}
function addHorarioPlantilla(dia,inicio,fin,materia,aula){
  form.value.horarios.push({ dia, inicio, fin, modalidad: 'Presencial', sinHorario: false, materia, programa: '', aula })
}
function addActividad(){
  form.value.actividadesLista.push({ nombre: '', horas: null, nota: '' })
}

/* === NUEVO: handlers para modalidad/checkbox === */
function onModalidadChange(h){
  if (h.modalidad === 'Presencial') {
    h.sinHorario = false
  } else if (h.modalidad === 'Virtual' && h.sinHorario) {
    // Virtual sin horario: limpia campos de día/hora
    h.dia = ''
    h.inicio = ''
    h.fin = ''
  }
}
function onSinHorarioToggle(h){
  if (h.sinHorario) {
    // al marcar "sin horario" limpiamos los campos
    h.dia = ''
    h.inicio = ''
    h.fin = ''
  }
}

function validateHorarios(list) {
  const warnings = []
  const groups = {}
  for (const h of list) {
    const isVirtual = h.modalidad === 'Virtual'
    const hasSchedule = !h.sinHorario && h.dia && h.inicio && h.fin

    // Virtual sin horario: permitido, no valida horas
    if (isVirtual && !hasSchedule && h.sinHorario) continue

    // Si no hay horario (faltan datos) y no es "sinHorario", pasa sin error
    if (!hasSchedule && isVirtual) continue

    const s = toMinutes(h.inicio), e = toMinutes(h.fin)
    if (hasSchedule && (s == null || e == null)) {
      warnings.push(`Formato inválido en ${h.dia} (${h.inicio}–${h.fin}).`)
      continue
    }
    if (hasSchedule && s != null && e != null && e <= s) {
      warnings.push(`Hora fin debe ser mayor que inicio en ${h.dia} (${h.inicio}–${h.fin}).`)
    }
    if (hasSchedule && s != null && e != null) {
      if (!groups[h.dia]) groups[h.dia] = []
      groups[h.dia].push([s,e,h])
    }
  }
  for (const dia in groups) {
    const arr = groups[dia].sort((a,b)=>a[0]-b[0])
    for (let i=1;i<arr.length;i++){
      const prev = arr[i-1], cur = arr[i]
      if (cur[0] < prev[1]) {
        const p = prev[2], c = cur[2]
        warnings.push(`Solape en ${dia}: "${p.materia||'Bloque'}" (${mins(prev[0])}–${mins(prev[1])}) con "${c.materia||'Bloque'}" (${mins(cur[0])}–${mins(cur[1])}).`)
      }
    }
  }
  return warnings
}

function save () {
  if (!form.value.cedula || !form.value.nombre || !form.value.tipo) {
    alert('Por favor completa CÉDULA, NOMBRE y TIPO.')
    return
  }
  const errs = validateHorarios(form.value.horarios || [])
  if (errs.length) {
    alert('Revisa los horarios:\n\n' + errs.join('\n'))
    return
  }
  form.value.cedula = String(form.value.cedula).trim()
  if (formMode.value === 'create') {
    const exists = profesores.value.some(p => String(p.cedula).trim() === form.value.cedula)
    if (exists) { alert('Ya existe un profesor con esa cédula.'); return }
    profesores.value.push(JSON.parse(JSON.stringify(form.value)))
    toast('Profesor creado')
  } else {
    const idx = profesores.value.findIndex(p => String(p.cedula).trim() === form.value.cedula)
    if (idx !== -1) profesores.value[idx] = JSON.parse(JSON.stringify(form.value))
    toast('Profesor actualizado')
  }
  saveDB()
  closeForm()
}
function removeTeacher (p) {
  if (!confirm(`¿Eliminar a ${p.nombre} (${p.cedula})?`)) return
  profesores.value = profesores.value.filter(x => x.cedula !== p.cedula)
  saveDB()
  toast('Profesor eliminado')
}
function verDetalle (p) {
  detalle.value = p
  showDetail.value = true
}

/* ===== Búsqueda rápida ===== */
function goQuickCedula () {
  if (!quickCedula.value) return
  const p = profesores.value.find(x => String(x.cedula).trim() === String(quickCedula.value).trim())
  if (p) { verDetalle(p); toast('Profesor encontrado') } else { toast('No existe esa cédula') }
}

/* ===== Impresión / Exportación ===== */
function imprimirUno (p) {
  const html = renderFichaHTML(p)
  const w = window.open('', '_blank')
  w.document.write(html)
  w.document.close()
  w.focus()
  w.print()
}
function printAll () {
  printList(filtrados.value)
}
function printList (list) {
  if (!list.length) { alert('No hay profesores para imprimir.'); return }
  const cards = list.map(p => renderFichaInnerHTML(p)).join('')
  const html = `<!DOCTYPE html><html lang="es"><head><meta charset="utf-8"/><title>Profesores</title>
  <style>${printStyles()}</style></head><body><h1>Profesores</h1><div class="grid">${cards}</div></body></html>`
  const w = window.open('', '_blank')
  w.document.write(html); w.document.close(); w.focus(); w.print()
}
function renderFichaHTML (p) {
  const inner = renderFichaInnerHTML(p)
  return `<!DOCTYPE html><html lang="es"><head><meta charset="utf-8"/><title>Profesor ${p.nombre}</title>
  <style>${printStyles()}</style></head><body>${inner}</body></html>`
}
function printStyles () {
  return `body{font-family:Inter,system-ui,Arial;padding:24px;max-width:980px;margin:0 auto;background:#fff}
  h1{margin:0 0 12px;font-size:20px}
  .grid{display:grid;grid-template-columns:repeat(auto-fill,minmax(280px,1fr));gap:16px}
  .card{border:1px solid #e5e7eb;border-radius:12px;padding:16px}
  .badge{display:inline-block;border:1px solid #e5e7eb;border-radius:9999px;padding:.125rem .5rem;font-size:12px;font-weight:600}
  .muted{color:#64748b;font-size:12px;margin-bottom:4px}
  table{width:100%;border-collapse:collapse} th,td{border:1px solid #e5e7eb;padding:6px;text-align:left} th{background:#f8fafc}`
}
function renderFichaInnerHTML (p) {
  const color = p.tipo === 'Planta' ? '#16a34a' : (p.tipo === 'Catedrático' ? '#d97706' : '#2563eb')
  const horarios = (p.horarios && p.horarios.length)
    ? `<div style="margin-top:12px">
        <div class="muted">Horarios</div>
        <table><thead><tr>
          <th>Día</th><th>Inicio</th><th>Fin</th><th>Materia</th><th>Programa</th><th>Aula</th><th>Modalidad</th>
        </tr></thead>
          <tbody>
            ${p.horarios.map(h => `<tr>
              <td>${(h.modalidad==='Virtual' && (h.sinHorario || !h.dia||!h.inicio||!h.fin)) ? '—' : (h.dia||'—')}</td>
              <td>${(h.modalidad==='Virtual' && (h.sinHorario || !h.dia||!h.inicio||!h.fin)) ? '—' : (h.inicio||'—')}</td>
              <td>${(h.modalidad==='Virtual' && (h.sinHorario || !h.dia||!h.inicio||!h.fin)) ? '—' : (h.fin||'—')}</td>
              <td>${h.materia||'—'}</td>
              <td>${h.programa||'—'}</td>
              <td>${h.aula||'—'}</td>
              <td>${h.modalidad}${h.modalidad==='Virtual' && h.sinHorario ? ' (sin horario)':''}</td>
            </tr>`).join('')}
          </tbody>
        </table>
      </div>` : ''
  const acts = Array.isArray(p.actividadesLista) && p.actividadesLista.length
    ? '<div class="muted">Actividades</div><ul>' + p.actividadesLista.map(a=>`<li>${a.nombre}${a.horas?` (${a.horas}h)`:''}${a.nota?` — ${a.nota}`:''}</li>`).join('') + '</ul>'
    : (p.actividades ? `<div class="muted">Actividades</div><div>${p.actividades.replace(/\n/g,'<br>')}</div>` : '')
  return `<div class="card">
    <div style="display:flex;justify-content:space-between;align-items:center;gap:8px">
      <div>
        <div class="muted">Cédula</div>
        <div style="font-family:ui-monospace,monospace;font-size:16px">${p.cedula}</div>
      </div>
      <span class="badge" style="color:${color}">${p.tipo}</span>
    </div>
    <h2 style="margin:6px 0 8px;font-size:18px">${p.nombre}</h2>
    <div style="display:grid;grid-template-columns:repeat(auto-fit,minmax(200px,1fr));gap:8px">
      <div><div class="muted">Horas semanales</div><div>${p.horasSemanales ?? '—'}</div></div>
      <div><div class="muted">Correo</div><div>${p.email||'—'}</div></div>
      <div><div class="muted">Ubicación de trabajo</div><div>${p.ubicacionTrabajo||'—'}</div></div>
      <div><div class="muted">Lugar de residencia</div><div>${p.lugarResidencia||'—'}</div></div>
      <div style="grid-column:1/-1">${acts}</div>
    </div>
    ${horarios}
  </div>`
}

/* ===== Selección (helpers) ===== */
function toggleSelectionMode () {
  selectionMode.value = !selectionMode.value
  if (!selectionMode.value) selected.value = new Set()
}
function isSelected (p) { return selected.value.has(String(p.cedula).trim()) }
function toggleSelect (p) {
  const key = String(p.cedula).trim()
  if (selected.value.has(key)) selected.value.delete(key)
  else selected.value.add(key)
  selected.value = new Set(selected.value) // trigger reactivity
}
function selectAllCurrent () {
  for (const p of filtrados.value) selected.value.add(String(p.cedula).trim())
  selected.value = new Set(selected.value)
}
function clearSelection () { selected.value = new Set() }
function toggleSelectAllVisible (e) {
  if (e && e.target && e.target.checked) selectAllCurrent()
  else {
    for (const p of filtrados.value) selected.value.delete(String(p.cedula).trim())
    selected.value = new Set(selected.value)
  }
}
function getSelectedList () {
  if (!selected.value.size) return filtrados.value.slice()
  return filtrados.value.filter(p => selected.value.has(String(p.cedula).trim()))
}
function printSelected () {
  const list = getSelectedList()
  printList(list)
}

/* ===== Exportar a PDF (html2pdf.js) ===== */
async function ensureHtml2Pdf () {
  if (window.html2pdf) return
  await new Promise((resolve, reject) => {
    const s = document.createElement('script')
    s.src = 'https://cdnjs.cloudflare.com/ajax/libs/html2pdf.js/0.10.1/html2pdf.bundle.min.js'
    s.onload = resolve; s.onerror = reject
    document.head.appendChild(s)
  })
}
async function exportSelectedPDF () {
  const list = getSelectedList()
  if (list.length === 0) { alert('No hay profesores para exportar.'); return }
  try {
    await ensureHtml2Pdf()
    const container = document.createElement('div')
    container.innerHTML = `<div style="font-family:Inter,Arial,sans-serif">${list.map(p => renderFichaInnerHTML(p)).join('')}</div>`
    const opt = {
      margin:       10,
      filename:     `profesores_${new Date().toISOString().slice(0,10)}.pdf`,
      image:        { type: 'jpeg', quality: 0.98 },
      html2canvas:  { scale: 2, useCORS: true },
      jsPDF:        { unit: 'mm', format: 'a4', orientation: 'portrait' }
    }
    await window.html2pdf().set(opt).from(container).save()
    toast('PDF exportado')
  } catch (e) {
    console.error(e)
    alert('No se pudo exportar a PDF. Usa "Imprimir" y elige Guardar como PDF.')
  }
}

/* ===== Import/Export JSON ===== */
function exportJSON () {
  const data = JSON.stringify(profesores.value, null, 2)
  const blob = new Blob([data], { type: 'application/json' })
  const url = URL.createObjectURL(blob)
  const a = document.createElement('a')
  const date = new Date().toISOString().slice(0, 10)
  a.href = url
  a.download = `profesores_${date}.json`
  a.click()
  URL.revokeObjectURL(url)
  toast('JSON exportado')
}
function handleImport (e) {
  const file = e.target.files?.[0]
  if (!file) return
  const reader = new FileReader()
  reader.onload = () => {
    try {
      const incoming = JSON.parse(reader.result)
      if (!Array.isArray(incoming)) throw new Error('El JSON debe ser un arreglo de profesores')
      const map = new Map(profesores.value.map(p => [String(p.cedula).trim(), p]))
      for (const p of incoming) {
        if (!p || !p.cedula) continue
        const key = String(p.cedula).trim()
        const safeHorarios = Array.isArray(p.horarios)
          ? p.horarios.map(normalizeHorario)
          : []
        map.set(key, {
          ...map.get(key),
          ...p,
          cedula: key,
          horarios: safeHorarios,
          actividadesLista: Array.isArray(p.actividadesLista) ? p.actividadesLista : (map.get(key)?.actividadesLista || []),
        })
      }
      profesores.value = Array.from(map.values())
      saveDB()
      toast('Importación completada')
    } catch (err) {
      console.error(err)
      alert('No se pudo importar el archivo. Verifica el formato JSON.')
    }
    e.target.value = ''
  }
  reader.readAsText(file)
}
</script>


<style scoped>
.shell{
  --bg: #f6f3ee;
  --bg-grad-1: #fffefb;
  --panel: #ffffff;
  --panel-2: #fdfcf9;
  --text: #2c2f2e;
  --muted: #6f7775;
  --border: #e7dfd4;
  --border-strong: #d8cfc2;
  --accent: #10b981;
  --accent-600: #0f9e73;
  --accent-50: #e9fbf4;
  --ok: #16a34a;
  --warn: #d97706;
  --danger: #dc2626;
  --radius-s: 12px;
  --radius-m: 16px;
  --radius-l: 20px;
  --shadow-s: 0 1px 2px rgba(30,35,34,.06);
  --shadow-m: 0 6px 18px rgba(30,35,34,.10);
  --shadow-l: 0 18px 44px rgba(30,35,34,.14);
}

* { box-sizing: border-box }
html { font-size: 16px }
body, .shell {
  background:
    radial-gradient(900px 600px at -10% -10%, var(--bg-grad-1) 0%, transparent 50%),
    radial-gradient(900px 600px at 120% 0%, var(--accent-50) 0%, transparent 50%),
    var(--bg);
  color: var(--text);
  -webkit-font-smoothing: antialiased; -moz-osx-font-smoothing: grayscale;
  font-family: Inter, ui-sans-serif, system-ui, -apple-system, Segoe UI, Roboto, Arial;
}
h1,h2,h3 { margin: 0; line-height: 1.25; font-weight: 800; letter-spacing: .2px }
p, label, input, select, textarea, button, table { font-size: 1rem; line-height: 1.5 }
a, button { font-weight: 600 }
.mono { font-family: ui-monospace, SFMono-Regular, Menlo, Monaco, Consolas, 'Liberation Mono', 'Courier New', monospace }
.semi { font-weight: 700 }
.muted { color: var(--muted) }
.text-right { text-align: right }
.req { color: var(--danger) }
.mb { margin-bottom: .5rem }
.mt { margin-top: 1rem }
.no-print {}
@media print { .no-print { display: none !important } }

/* Layout */
.shell { min-height: 100vh; display: grid; grid-template-rows: auto auto auto 1fr; gap: 14px; padding-bottom: 16px }

/* Topbar */
.nav{
  display:flex; align-items:center; justify-content:space-between; gap:16px;
  padding: 16px 18px; background: var(--panel); border: 1px solid var(--border);
  border-radius: var(--radius-l); box-shadow: var(--shadow-m); margin: 16px 16px 0 16px;
}
.brand{ display:flex; align-items:center; gap:12px }
.logo{
  width:48px; height:48px; display:grid; place-items:center; border-radius: 14px;
  background: linear-gradient(135deg, #b7f1d9, var(--accent));
  color: #083b2d; font-weight: 900; box-shadow: var(--shadow-s);
}
.title{ font-weight: 900; letter-spacing:.2px; font-size: 1.15rem }
.subtitle{ font-size:.8rem; color: var(--muted) }
.brand-text{ display:grid; gap:2px }
.actions{ display:flex; gap:10px; flex-wrap:wrap; align-items:center }

.btn{
  border: 1.5px solid var(--border);
  background: var(--panel);
  color: var(--text);
  padding: 10px 14px;
  border-radius: var(--radius-s);
  cursor: pointer;
  transition: .15s;
  box-shadow: var(--shadow-s);
}
.btn:hover{ box-shadow: var(--shadow-m); transform: translateY(-1px) }
.btn:focus-visible{ outline: 3px solid #c7f4e6; outline-offset: 2px }
.btn.primary{ color:#07372a; background: linear-gradient(135deg, #7ee6c4, var(--accent)); border-color: transparent }
.btn.primary:hover{ filter: brightness(1.03) }
.btn.danger{ color:#fff; background: #e26363; border-color: #e26363 }
.btn.subtle{ background:#faf7f2; border-color:#e9e2d8 }
.icon-btn{ border: 1.5px solid var(--border); background: var(--panel); color: var(--text); padding: 6px 10px; border-radius: var(--radius-s); cursor: pointer }
.link{ background: transparent; border: 0; color: #0f9e73; cursor: pointer; padding: 6px 8px; border-radius: 8px }
.link:hover{ background: #e8fff7 }
.link.danger{ color: #d04b4b }
label.btn { position: relative; overflow: hidden }
label.btn input[type="file"]{ position:absolute; inset:0; opacity:0; cursor:pointer }

/* Filtros */
.filters{
  display:grid; grid-template-columns: 2fr 2fr 1.2fr auto auto; gap:12px;
  margin: 0 16px; padding: 16px 18px; background: var(--panel);
  border: 1px solid var(--border); border-radius: var(--radius-l); box-shadow: var(--shadow-m);
  align-items: end;
}
.field{ display:grid; gap:6px }
.field.wide{ grid-column: span 2 }
.field.compact{ width: 220px }
.field label{ font-size:.875rem; color: var(--muted); font-weight: 600; letter-spacing:.2px }
.field input, .field select, .field textarea{
  border: 1.5px solid var(--border); border-radius: var(--radius-s);
  padding: 10px 12px; outline: none; background: var(--panel-2); color: var(--text);
}
.field textarea{ resize: vertical }
.field input::placeholder, .field textarea::placeholder { color:#9aa3b2 }
.field input:focus, .field select:focus, .field textarea:focus{ border-color: var(--accent); box-shadow: 0 0 0 3px #c8efdf }
.field input:disabled, .field select:disabled{ background:#f3f0ea; color:#999; cursor:not-allowed; border-color:#e8e1d7 }

.row{ display:flex; gap:8px }
.segmented{ background: #f2efe8; border: 1px solid var(--border); border-radius: 999px; padding: 4px; display:flex; gap:4px }
.segmented button{ border:0; background: transparent; border-radius: 999px; padding: 6px 12px; cursor:pointer; color: var(--muted) }
.segmented button.on{ background:#fff; color: var(--text); border:1px solid var(--border); box-shadow: var(--shadow-s) }

.counter{ justify-self:end; align-self:center }
.chip{ display:inline-block; padding: 6px 10px; border-radius: 999px; background:#e8fff7; color:#065f46; border: 1px solid #b9f3df; font-weight:700; font-size:.85rem }

/* Bulk actions */
.bulkbar{
  margin: 0 16px;
  background: var(--panel);
  border: 1px solid var(--border);
  border-radius: var(--radius-l);
  box-shadow: var(--shadow-m);
  padding: 10px 12px;
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 12px;
}
.bulk-left{ font-weight: 800 }
.bulk-right{ display: flex; gap: 8px; flex-wrap: wrap }

/* Cards */
.cards{ padding: 2px 18px 18px; display:grid; grid-template-columns: repeat(auto-fill,minmax(320px,1fr)); gap: 18px }
.empty{ grid-column: 1 / -1; text-align:center; color: var(--muted); padding: 28px; border: 2px dashed var(--border); border-radius: var(--radius-m); background: #faf7f2 }
.card{ position: relative; background: var(--panel); border: 1px solid var(--border); border-radius: var(--radius-l); padding: 14px 16px; box-shadow: var(--shadow-m); transition:.18s }
.card:hover{ transform: translateY(-2px) }
.card-bar{ content:""; position:absolute; left:0; top:0; right:0; height:8px; border-top-left-radius: var(--radius-l); border-top-right-radius: var(--radius-l); background: linear-gradient(90deg, #bff1dd, #a7edd3 40%, #8ee8c8) }
.card-head{ display:flex; justify-content:space-between; align-items:center; gap:8px; margin-top: 4px; margin-bottom: 10px }
.person{ display:flex; gap:10px; align-items:center }
.avatar{ width: 44px; height: 44px; display:grid; place-items:center; border-radius: 12px; background: #e9fbf4; color:#065f46; font-weight:800 }
.name{ font-weight:800; letter-spacing:.2px; font-size: 1.06rem }
.id{ font-size:.875rem; color: var(--muted) }
.grid{ display:grid; grid-template-columns: 1fr 1fr; gap:8px; margin-top:6px }
.activities{ margin-top: 8px; display:flex; gap: 6px; flex-wrap: wrap }
.act-chip{ background:#eefcf7; color:#0b4e3c; border:1px solid #c9f0e0; padding: 4px 8px; border-radius: 999px; font-size:.8rem }
.act-note{ color: var(--muted); font-size: .95rem }
.card-actions{ display:flex; justify-content:flex-end; gap:10px; margin-top: 12px }
.pill{ display:inline-block; padding: 4px 10px; font-size:.8rem; font-weight:800; border-radius:999px; border:1px solid var(--border); letter-spacing:.2px }
.pill--planta{ background:#ecfdf5; color:#166534; border-color:#a7f3d0 }
.pill--catedra{ background:#fff7e6; color:#7c4a0e; border-color:#f8e0b3 }
.pill--contrato{ background:#eef6ff; color:#1e40af; border-color:#bfdbfe }
.pill--neutral{ background:#f1f5f9; color:#334155 }

/* Card selection checkbox */
.select-box{
  position: absolute; top: 10px; left: 10px;
  background: #fff; border: 1px solid var(--border); border-radius: 10px;
  padding: 4px; box-shadow: var(--shadow-s);
}
.select-box input{ width: 18px; height: 18px }

/* Tabla */
.table-card{ padding: 0 18px 18px }
.table-wrapper{ overflow:auto; border-radius: var(--radius-m); border: 1px solid var(--border); background: var(--panel); box-shadow: var(--shadow-m) }
.table{ width:100%; border-collapse: collapse; font-size: 0.975rem }
.table th, .table td{ padding: 12px; border-bottom: 1px solid var(--border) }
.table thead th{ text-align:left; color: var(--muted); background:#faf7f2; position: sticky; top: 0; z-index: 1; border-bottom:1px solid var(--border-strong) }
.table tbody tr:nth-child(odd){ background: #fffefb }
.table tbody tr:hover{ background: #f6fff9 }
.table .empty{ text-align:center; color: var(--muted) }
.row-actions{ display:flex; justify-content:flex-end; gap:8px }
.table input[type="checkbox"]{ width: 18px; height: 18px; accent-color: var(--accent) }

/* Planner */
.planner{ padding: 0 18px 18px; display:grid; gap:12px }
.planner-head{ display:flex; align-items:center; justify-content:space-between; background: var(--panel); border:1px solid var(--border); border-radius: var(--radius-m); padding: 10px 12px; box-shadow: var(--shadow-s) }
.legend{ display:flex; gap:8px; flex-wrap:wrap }
.legend-item{ padding: 4px 10px; border-radius: 999px; font-size:.8rem; border:1px solid var(--border); font-weight:700; background:#fff }
.legend-item.planta{ background:#ecfdf5; color:#166534; border-color:#a7f3d0 }
.legend-item.catedra{ background:#fff7e6; color:#7c4a0e; border-color:#f8e0b3 }
.legend-item.contrato{ background:#eef6ff; color:#1e40af; border-color:#bfdbfe }
.legend-item.mode{ background:#eefcf7; color:#0b4e3c; border-color:#c9f0e0 }

.planner-grid{ display:grid; grid-template-columns: 84px 1fr; gap:12px }
.hours-rail{ background: var(--panel); border:1px solid var(--border); border-radius: var(--radius-m); padding: 8px; box-shadow: var(--shadow-s) }
.hour{ height: 60px; display:flex; align-items:flex-start; justify-content:flex-end; padding-right:6px; color: var(--muted); font-size:.85rem }

.days{ position:relative; background: var(--panel); border:1px solid var(--border); border-radius: var(--radius-m); overflow:auto; padding-top: 34px; box-shadow: var(--shadow-s) }
.day-header{ position: sticky; top: 0; height: 34px; display:inline-flex; align-items:center; justify-content:center; color:#2c2f2e; font-weight:700; font-size:.85rem; border-right:1px solid var(--border); width: calc(100%/7); background:#f9f7f3; float:left }
.day-col{ position:relative; width: calc(100%/7); float:left; border-right:1px dashed #eadfce; background-image: repeating-linear-gradient(to bottom, #f2eee7 0 1px, transparent 1px 60px) }
.hour-line{ display:none }

.block{ position:absolute; left:6px; right:6px; border-radius: 12px; padding: 6px 8px; color:#1f2937; font-size:.85rem; overflow:hidden; border: 1px solid #e3d9c9; box-shadow: var(--shadow-s) }
.block .b-title{ font-weight:800 }
.block .b-sub{ color: var(--muted); font-weight:600 }
.k-planta{ background:#d7f7eb }
.k-catedra{ background:#fff0d6 }
.k-contrato{ background:#e4efff }
.block.m-virtual{ border-style: dashed }
.block.m-presencial{ border-style: solid }

/* Virtuales sin horario */
.virtual-panel{ background:#ffffff; border:1px solid var(--border); border-radius: var(--radius-m); box-shadow: var(--shadow-s); padding: 12px 14px }
.virtual-head{ font-weight:800; margin-bottom:8px }
.virtual-list{ list-style:none; padding:0; margin:0; display:grid; gap:6px }
.virtual-list li{ display:flex; flex-wrap:wrap; gap:6px; align-items:center }
.v-materia{ font-weight:800 }
.v-sep{ color: var(--muted) }
.v-prof{ color:#0f9e73; font-weight:700 }
.v-prog{ color:#374151; font-weight:600 }
.v-aula{ color:#6b7280 }

/* Toasts & Modals */
.toasts{ position: fixed; right: 16px; bottom: 16px; display:grid; gap:8px; z-index:60 }
.toast{ background:#ffffff; color:#2c2f2e; border:1.5px solid var(--border); padding: 10px 12px; border-radius: var(--radius-s); box-shadow: var(--shadow-l) }

.overlay{ position: fixed; inset: 0; display:grid; place-items:center; background: rgba(22, 24, 23, .55); padding:16px; z-index:1000 }
.modal{ width: min(980px, 96vw); background: #ffffff; border: 1px solid var(--border-strong); border-radius: var(--radius-l); padding: 16px; box-shadow: var(--shadow-l); max-height: 92vh; overflow: auto }
.modal-head{ display:flex; align-items:start; justify-content:space-between; gap:12px; margin-bottom:12px }
.form-grid{ display:grid; grid-template-columns: 1fr 1fr; gap: 12px }
.form-grid .col-2{ grid-column: 1 / -1 }
.subhead{ display:flex; align-items:center; justify-content:space-between; gap:8px; margin-top: 2px }
.sub-actions{ display:flex; gap:6px; flex-wrap:wrap }

/* Horario row (con Programa) */
.horario{
  display:grid; grid-template-columns: 140px 110px 110px 140px 1fr 1fr 1fr;
  gap:8px; margin-top: 8px;
}
.horario-right{ display:flex; gap:8px }

/* Detalle / impresión */
.pre{ white-space: pre-wrap }
.warnings{ margin-top:8px; background:#fff7e6; border:1px solid #f8e0b3; color:#7c4a0e; border-radius: 10px; padding:8px }
.detail .detail-top{ display:flex; justify-content:space-between; align-items:center }
.detail .detail-name{ margin: 6px 0 10px; font-size: 1.12rem; font-weight: 900 }
.detail .detail-grid{ display:grid; grid-template-columns: 1fr 1fr; gap: 12px }
.print-card{ break-inside: avoid; page-break-inside: avoid; border: 1px solid var(--border); border-radius: var(--radius-m); padding: 12px }

/* Responsive */
@media (max-width: 980px){
  .filters{ grid-template-columns: 1fr }
  .grid{ grid-template-columns: 1fr }
  .form-grid{ grid-template-columns: 1fr }
  .horario{ grid-template-columns: 1fr 1fr }
  .days, .hours-rail{ max-height: 60vh }
}

/* Print */
@media print{
  .shell{ background:#fff; color:#000 }
  .table thead th{ background:#fff }
  .no-print{ display:none !important }
}
</style>
