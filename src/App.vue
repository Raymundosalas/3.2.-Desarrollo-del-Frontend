
<script setup>
import { ref, computed, onMounted } from 'vue';
import api from './services/api';

const activeTab = ref('inicio');

const teams = ref([]);
const players = ref([]);
const matches = ref([]);

const loading = ref(false);
const saving = ref(false);
const error = ref('');
const message = ref('');

const teamForm = ref({
  name: '',
  city: '',
  coach: '',
  category: ''
});

const playerForm = ref({
  team_id: '',
  first_name: '',
  last_name: '',
  jersey_number: '',
  position: ''
});

const matchForm = ref({
  home_team_id: '',
  away_team_id: '',
  scheduled_at: '',
  venue: '',
  home_score: 0,
  away_score: 0,
  status: 'scheduled'
});

const navigation = [
  { id: 'inicio', label: 'Inicio', icon: '⌂' },
  { id: 'equipos', label: 'Equipos', icon: '⚾' },
  { id: 'jugadores', label: 'Jugadores', icon: '♟' },
  { id: 'partidos', label: 'Partidos', icon: '▣' }
];

const pageTitle = computed(() => {
  const item = navigation.find(option => option.id === activeTab.value);
  return item ? item.label : 'Inicio';
});

const upcomingMatches = computed(() => {
  return [...matches.value].sort((a, b) => {
    return new Date(a.scheduled_at) - new Date(b.scheduled_at);
  }).slice(0, 5);
});

function showMessage(text) {
  message.value = text;
  error.value = '';

  setTimeout(() => {
    message.value = '';
  }, 3500);
}

function showError(err) {
  const response = err.response;

  if (response?.data?.errors) {
    const validationErrors = Object.values(response.data.errors).flat();
    error.value = validationErrors.join(' ');
  } else {
    error.value = response?.data?.message ||
      'No fue posible completar la operación. Verifica la conexión con Laravel.';
  }

  message.value = '';
}

async function loadAll() {
  loading.value = true;
  error.value = '';

  try {
    const [teamResponse, playerResponse, matchResponse] =
      await Promise.all([
        api.get('/teams'),
        api.get('/players'),
        api.get('/matches')
      ]);

    teams.value = Array.isArray(teamResponse.data)
      ? teamResponse.data
      : teamResponse.data.data || [];

    players.value = Array.isArray(playerResponse.data)
      ? playerResponse.data
      : playerResponse.data.data || [];

    matches.value = Array.isArray(matchResponse.data)
      ? matchResponse.data
      : matchResponse.data.data || [];

  } catch (err) {
    showError(err);
  } finally {
    loading.value = false;
  }
}

async function createTeam() {
  saving.value = true;

  try {
    await api.post('/teams', teamForm.value);

    teamForm.value = {
      name: '',
      city: '',
      coach: '',
      category: ''
    };

    await loadAll();
    showMessage('El equipo se registró correctamente.');
  } catch (err) {
    showError(err);
  } finally {
    saving.value = false;
  }
}

async function deleteTeam(id) {
  if (!window.confirm('¿Deseas eliminar este equipo?')) return;

  try {
    await api.delete(`/teams/${id}`);
    await loadAll();
    showMessage('Equipo eliminado correctamente.');
  } catch (err) {
    showError(err);
  }
}

async function createPlayer() {
  saving.value = true;

  try {
    await api.post('/players', {
      ...playerForm.value,
      team_id: Number(playerForm.value.team_id),
      jersey_number: Number(playerForm.value.jersey_number)
    });

    playerForm.value = {
      team_id: '',
      first_name: '',
      last_name: '',
      jersey_number: '',
      position: ''
    };

    await loadAll();
    showMessage('El jugador se registró correctamente.');
  } catch (err) {
    showError(err);
  } finally {
    saving.value = false;
  }
}

async function deletePlayer(id) {
  if (!window.confirm('¿Deseas eliminar este jugador?')) return;

  try {
    await api.delete(`/players/${id}`);
    await loadAll();
    showMessage('Jugador eliminado correctamente.');
  } catch (err) {
    showError(err);
  }
}

async function createMatch() {
  if (
    Number(matchForm.value.home_team_id) ===
    Number(matchForm.value.away_team_id)
  ) {
    error.value = 'El equipo local y visitante deben ser diferentes.';
    return;
  }

  saving.value = true;

  try {
    await api.post('/matches', {
      ...matchForm.value,
      home_team_id: Number(matchForm.value.home_team_id),
      away_team_id: Number(matchForm.value.away_team_id),
      home_score: Number(matchForm.value.home_score),
      away_score: Number(matchForm.value.away_score)
    });

    matchForm.value = {
      home_team_id: '',
      away_team_id: '',
      scheduled_at: '',
      venue: '',
      home_score: 0,
      away_score: 0,
      status: 'scheduled'
    };

    await loadAll();
    showMessage('El partido se registró correctamente.');
  } catch (err) {
    showError(err);
  } finally {
    saving.value = false;
  }
}

async function deleteMatch(id) {
  if (!window.confirm('¿Deseas eliminar este partido?')) return;

  try {
    await api.delete(`/matches/${id}`);
    await loadAll();
    showMessage('Partido eliminado correctamente.');
  } catch (err) {
    showError(err);
  }
}

function teamName(id) {
  const team = teams.value.find(item => item.id === id);
  return team?.name || 'Equipo no disponible';
}

function formatDate(value) {
  if (!value) return 'Sin fecha';

  const date = new Date(value);

  if (Number.isNaN(date.getTime())) return value;

  return date.toLocaleString('es-MX', {
    dateStyle: 'medium',
    timeStyle: 'short'
  });
}

function statusLabel(status) {
  const labels = {
    scheduled: 'Programado',
    in_progress: 'En curso',
    completed: 'Finalizado',
    cancelled: 'Cancelado'
  };

  return labels[status] || status;
}

onMounted(loadAll);
</script>

<template>
  <div class="app-layout">

    <aside class="sidebar">
      <div class="brand">
        <div class="brand-icon">⚾</div>
        <div>
          <h2>LIGA BEISBOL</h2>
          <span>Administración deportiva</span>
        </div>
      </div>

      <div class="menu-label">MENÚ PRINCIPAL</div>

      <nav class="navigation">
        <button
          v-for="item in navigation"
          :key="item.id"
          :class="['nav-item', { active: activeTab === item.id }]"
          @click="activeTab = item.id"
        >
          <span class="nav-icon">{{ item.icon }}</span>
          {{ item.label }}
        </button>
      </nav>

      <div class="sidebar-footer">
        <span class="status-dot"></span>
        Sistema de administración
      </div>
    </aside>

    <main class="main-content">

      <header class="topbar">
        <div>
          <span class="breadcrumb">Sistema / {{ pageTitle }}</span>
          <h1>{{ pageTitle }}</h1>
        </div>

        <button class="refresh-button" @click="loadAll">
          ↻ Actualizar información
        </button>
      </header>

      <div class="content-area">

        <div v-if="message" class="alert success">
          {{ message }}
        </div>

        <div v-if="error" class="alert error">
          {{ error }}
        </div>

        <div v-if="loading" class="loading">
          Cargando información...
        </div>

        <!-- INICIO -->
        <section v-if="activeTab === 'inicio'">

          <div class="welcome">
            <div>
              <span class="welcome-tag">PANEL ADMINISTRATIVO</span>
              <h2>Bienvenido al sistema de la liga</h2>
              <p>
                Administra equipos, jugadores y partidos desde una sola
                plataforma.
              </p>
            </div>
            <div class="welcome-ball">⚾</div>
          </div>

          <div class="stats-grid">

            <article class="stat-card">
              <div class="stat-icon green">⚾</div>
              <span>Equipos registrados</span>
              <strong>{{ teams.length }}</strong>
              <small>Equipos de la liga</small>
            </article>

            <article class="stat-card">
              <div class="stat-icon blue">♟</div>
              <span>Jugadores registrados</span>
              <strong>{{ players.length }}</strong>
              <small>Participantes registrados</small>
            </article>

            <article class="stat-card">
              <div class="stat-icon orange">▣</div>
              <span>Partidos registrados</span>
              <strong>{{ matches.length }}</strong>
              <small>Encuentros de la liga</small>
            </article>

          </div>

          <section class="panel">
            <div class="panel-heading">
              <div>
                <h3>Próximos partidos</h3>
                <p>Consulta los encuentros registrados.</p>
              </div>
              <button class="text-button" @click="activeTab = 'partidos'">
                Ver todos
              </button>
            </div>

            <div v-if="upcomingMatches.length" class="table-container">
              <table>
                <thead>
                  <tr>
                    <th>Equipo local</th>
                    <th>Equipo visitante</th>
                    <th>Fecha</th>
                    <th>Estado</th>
                  </tr>
                </thead>
                <tbody>
                  <tr v-for="match in upcomingMatches" :key="match.id">
                    <td>{{ match.home_team?.name || teamName(match.home_team_id) }}</td>
                    <td>{{ match.away_team?.name || teamName(match.away_team_id) }}</td>
                    <td>{{ formatDate(match.scheduled_at) }}</td>
                    <td>
                      <span class="status-badge">
                        {{ statusLabel(match.status) }}
                      </span>
                    </td>
                  </tr>
                </tbody>
              </table>
            </div>

            <div v-else class="empty-state">
              Todavía no hay partidos registrados.
            </div>
          </section>

        </section>

        <!-- EQUIPOS -->
        <section v-if="activeTab === 'equipos'">

          <div class="section-intro">
            <div>
              <h2>Administración de equipos</h2>
              <p>Registra y consulta los equipos participantes.</p>
            </div>
            <span class="counter">{{ teams.length }} equipos</span>
          </div>

          <section class="panel">
            <div class="panel-heading">
              <div>
                <h3>Registrar equipo</h3>
                <p>Completa los datos del nuevo equipo.</p>
              </div>
            </div>

            <form @submit.prevent="createTeam" class="form-grid">

              <label>
                Nombre del equipo
                <input v-model.trim="teamForm.name" required
                  placeholder="Ej. Halcones">
              </label>

              <label>
                Ciudad
                <input v-model.trim="teamForm.city" required
                  placeholder="Ej. Guadalajara">
              </label>

              <label>
                Entrenador
                <input v-model.trim="teamForm.coach"
                  placeholder="Nombre del entrenador">
              </label>

              <label>
                Categoría
                <input v-model.trim="teamForm.category" required
                  placeholder="Ej. Libre, Juvenil">
              </label>

              <div class="form-actions">
                <button class="primary-button" type="submit" :disabled="saving">
                  {{ saving ? 'Guardando...' : '+ Registrar equipo' }}
                </button>
              </div>

            </form>
          </section>

          <section class="panel">
            <div class="panel-heading">
              <div>
                <h3>Equipos registrados</h3>
                <p>Información almacenada en el sistema.</p>
              </div>
            </div>

            <div v-if="teams.length" class="table-container">
              <table>
                <thead>
                  <tr>
                    <th>Nombre</th>
                    <th>Ciudad</th>
                    <th>Entrenador</th>
                    <th>Categoría</th>
                    <th>Acciones</th>
                  </tr>
                </thead>

                <tbody>
                  <tr v-for="team in teams" :key="team.id">
                    <td><strong>{{ team.name }}</strong></td>
                    <td>{{ team.city }}</td>
                    <td>{{ team.coach || 'No especificado' }}</td>
                    <td>{{ team.category }}</td>
                    <td>
                      <button class="delete-button"
                        @click="deleteTeam(team.id)">
                        Eliminar
                      </button>
                    </td>
                  </tr>
                </tbody>
              </table>
            </div>

            <div v-else class="empty-state">
              No existen equipos registrados.
            </div>
          </section>

        </section>

        <!-- JUGADORES -->
        <section v-if="activeTab === 'jugadores'">

          <div class="section-intro">
            <div>
              <h2>Administración de jugadores</h2>
              <p>Registra a los participantes y asígnalos a un equipo.</p>
            </div>
            <span class="counter">{{ players.length }} jugadores</span>
          </div>

          <section class="panel">
            <div class="panel-heading">
              <div>
                <h3>Registrar jugador</h3>
                <p>Introduce los datos del participante.</p>
              </div>
            </div>

            <form @submit.prevent="createPlayer" class="form-grid">

              <label>
                Nombre
                <input v-model.trim="playerForm.first_name" required
                  placeholder="Nombre del jugador">
              </label>

              <label>
                Apellido
                <input v-model.trim="playerForm.last_name" required
                  placeholder="Apellido del jugador">
              </label>

              <label>
                Número de camiseta
                <input v-model.number="playerForm.jersey_number"
                  type="number" min="0" max="99" required>
              </label>

              <label>
                Posición
                <input v-model.trim="playerForm.position" required
                  placeholder="Ej. Jardinero">
              </label>

              <label>
                Equipo
                <select v-model="playerForm.team_id" required>
                  <option value="" disabled>Selecciona un equipo</option>
                  <option v-for="team in teams" :key="team.id"
                    :value="team.id">
                    {{ team.name }}
                  </option>
                </select>
              </label>

              <div class="form-actions">
                <button class="primary-button" type="submit"
                  :disabled="saving || teams.length === 0">
                  {{ saving ? 'Guardando...' : '+ Registrar jugador' }}
                </button>
              </div>

              <p v-if="teams.length === 0" class="help-text">
                Primero debes registrar un equipo.
              </p>

            </form>
          </section>

          <section class="panel">
            <div class="panel-heading">
              <div>
                <h3>Jugadores registrados</h3>
                <p>Consulta los participantes de la liga.</p>
              </div>
            </div>

            <div v-if="players.length" class="table-container">
              <table>
                <thead>
                  <tr>
                    <th>Nombre</th>
                    <th>Equipo</th>
                    <th>Número</th>
                    <th>Posición</th>
                    <th>Acciones</th>
                  </tr>
                </thead>

                <tbody>
                  <tr v-for="player in players" :key="player.id">
                    <td>
                      <strong>
                        {{ player.first_name }} {{ player.last_name }}
                      </strong>
                    </td>
                    <td>{{ player.team?.name || teamName(player.team_id) }}</td>
                    <td>#{{ player.jersey_number }}</td>
                    <td>{{ player.position }}</td>
                    <td>
                      <button class="delete-button"
                        @click="deletePlayer(player.id)">
                        Eliminar
                      </button>
                    </td>
                  </tr>
                </tbody>
              </table>
            </div>

            <div v-else class="empty-state">
              No existen jugadores registrados.
            </div>
          </section>

        </section>

        <!-- PARTIDOS -->
        <section v-if="activeTab === 'partidos'">

          <div class="section-intro">
            <div>
              <h2>Administración de partidos</h2>
              <p>Registra encuentros y consulta sus resultados.</p>
            </div>
            <span class="counter">{{ matches.length }} partidos</span>
          </div>

          <section class="panel">
            <div class="panel-heading">
              <div>
                <h3>Registrar partido</h3>
                <p>Define los equipos y los datos del encuentro.</p>
              </div>
            </div>

            <form @submit.prevent="createMatch" class="form-grid">

              <label>
                Equipo local
                <select v-model="matchForm.home_team_id" required>
                  <option value="" disabled>Selecciona un equipo</option>
                  <option v-for="team in teams" :key="team.id"
                    :value="team.id">
                    {{ team.name }}
                  </option>
                </select>
              </label>

              <label>
                Equipo visitante
                <select v-model="matchForm.away_team_id" required>
                  <option value="" disabled>Selecciona un equipo</option>
                  <option v-for="team in teams" :key="team.id"
                    :value="team.id">
                    {{ team.name }}
                  </option>
                </select>
              </label>

              <label>
                Fecha y hora
                <input v-model="matchForm.scheduled_at"
                  type="datetime-local" required>
              </label>

              <label>
                Campo o sede
                <input v-model.trim="matchForm.venue" required
                  placeholder="Ej. Campo municipal">
              </label>

              <label>
                Marcador local
                <input v-model.number="matchForm.home_score"
                  type="number" min="0" required>
              </label>

              <label>
                Marcador visitante
                <input v-model.number="matchForm.away_score"
                  type="number" min="0" required>
              </label>

              <label>
                Estado del partido
                <select v-model="matchForm.status" required>
                  <option value="scheduled">Programado</option>
                  <option value="in_progress">En curso</option>
                  <option value="completed">Finalizado</option>
                  <option value="cancelled">Cancelado</option>
                </select>
              </label>

              <div class="form-actions">
                <button class="primary-button" type="submit"
                  :disabled="saving || teams.length < 2">
                  {{ saving ? 'Guardando...' : '+ Registrar partido' }}
                </button>
              </div>

              <p v-if="teams.length < 2" class="help-text">
                Necesitas registrar al menos dos equipos para crear un partido.
              </p>

            </form>
          </section>

          <section class="panel">
            <div class="panel-heading">
              <div>
                <h3>Partidos registrados</h3>
                <p>Encuentros y resultados de la liga.</p>
              </div>
            </div>

            <div v-if="matches.length" class="table-container">
              <table>
                <thead>
                  <tr>
                    <th>Equipo local</th>
                    <th>Marcador</th>
                    <th>Equipo visitante</th>
                    <th>Fecha</th>
                    <th>Sede</th>
                    <th>Estado</th>
                    <th>Acciones</th>
                  </tr>
                </thead>

                <tbody>
                  <tr v-for="match in matches" :key="match.id">
                    <td>
                      {{ match.home_team?.name || teamName(match.home_team_id) }}
                    </td>
                    <td>
                      <strong>
                        {{ match.home_score }} - {{ match.away_score }}
                      </strong>
                    </td>
                    <td>
                      {{ match.away_team?.name || teamName(match.away_team_id) }}
                    </td>
                    <td>{{ formatDate(match.scheduled_at) }}</td>
                    <td>{{ match.venue }}</td>
                    <td>
                      <span class="status-badge">
                        {{ statusLabel(match.status) }}
                      </span>
                    </td>
                    <td>
                      <button class="delete-button"
                        @click="deleteMatch(match.id)">
                        Eliminar
                      </button>
                    </td>
                  </tr>
                </tbody>
              </table>
            </div>

            <div v-else class="empty-state">
              No existen partidos registrados.
            </div>
          </section>

        </section>

      </div>

      <footer class="footer">
        Sistema Web para la Gestión y Administración de una Liga de Béisbol
        <span>Desarrollado con Vue.js y Laravel</span>
      </footer>

    </main>
  </div>
</template>
