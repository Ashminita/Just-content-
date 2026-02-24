
const Route = sequelize.define("Route", {
  route_id: {
    type: DataTypes.INTEGER,
    primaryKey: true,
    autoIncrement: true,
  },
  distance: {
    type: DataTypes.FLOAT,
  },
});

Route.belongsTo(Airport, {
  as: "departure_airport",
  foreignKey: "departure_airport_code",
});

Route.belongsTo(Airport, {
  as: "arrival_airport",
  foreignKey: "arrival_airport_code",
});

module.exports = Route;
4️⃣ Flight Model
📁 models/Flight.js
JavaScript
Copy code
const { DataTypes } = require("sequelize");
const sequelize = require("./index");
const Aircraft = require("./Aircraft");
const Route = require("./Route");

const Flight = sequelize.define("Flight", {
  flight_id: {
    type: DataTypes.INTEGER,
    primaryKey: true,
    autoIncrement: true,
  },
  flight_number: {
    type: DataTypes.STRING,
    allowNull: false,
  },
  departure_date: {
    type: DataTypes.DATEONLY,
  },
  departure_time: {
    type: DataTypes.TIME,
  },
  arrival_time: {
    type: DataTypes.TIME,
  },
  status: {
    type: DataTypes.ENUM("Scheduled", "Delayed", "Cancelled", "Completed"),
    defaultValue: "Scheduled",
  },
});

Flight.belongsTo(Aircraft, {
  foreignKey: "aircraft_id",
});

Flight.belongsTo(Route, {
  foreignKey: "route_id",
});

module.exports = Flight;
5️⃣ Crew Model
📁 models/Crew.js
JavaScript
Copy code
const { DataTypes } = require("sequelize");
const sequelize = require("./index");

const Crew = sequelize.define("Crew", {
  crew_id: {
    type: DataTypes.INTEGER,
    primaryKey: true,
    autoIncrement: true,
  },
  name: {
    type: DataTypes.STRING,
    allowNull: false,
  },
  role: {
    type: DataTypes.ENUM("Pilot", "Co-Pilot", "Cabin Crew"),
  },
  contact: {
    type: DataTypes.STRING,
  },
  license_number: {
    type: DataTypes.STRING,
  },
});

module.exports = Crew;
6️⃣ FlightCrew (Junction Table)
📁 models/FlightCrew.js
JavaScript
Copy code
const sequelize = require("./index");
const Flight = require("./Flight");
const Crew = require("./Crew");

const FlightCrew = sequelize.define("FlightCrew", {}, { timestamps: false });

Flight.belongsToMany(Crew, {
  through: FlightCrew,
  foreignKey: "flight_id",
});

Crew.belongsToMany(Flight, {
  through: FlightCrew,
  foreignKey: "crew_id",
});

module.exports = FlightCrew;
7️⃣ Maintenance Model
📁 models/Maintenance.js
JavaScript
Copy code
const { DataTypes } = require("sequelize");
const sequelize = require("./index");
const Aircraft = require("./Aircraft");

const Maintenance = sequelize.define("Maintenance", {
  maintenance_id: {
    type: DataTypes.INTEGER,
    primaryKey: true,
    autoIncrement: true,
  },
  maintenance_date: {
    type: DataTypes.DATEONLY,
  },
  maintenance_type: {
    type: DataTypes.STRING,
  },
  status: {
    type: DataTypes.STRING,
  },
  technician_name: {
    type: DataTypes.STRING,
  },
});

Maintenance.belongsTo(Aircraft, {
  foreignKey: "aircraft_id",
});

module.exports = Maintenance;
🔗 Final Relationships (Clear Understanding)
Aircraft → Flight (1:M)
Airport → Route (1:M)
Route → Flight (1:M)
Flight ↔ Crew (M:M via FlightCrew)
Aircraft → Maintenance (1:M)

Flight Service (Business Logic)
📁 services/flight.service.js
JavaScript
Copy code
const Flight = require("../models/Flight");
const Aircraft = require("../models/Aircraft");
const Route = require("../models/Route");

// Create Flight
exports.createFlight = async (data) => {
  const { aircraft_id, route_id, departure_time } = data;

  // 1. Check aircraft exists
  const aircraft = await Aircraft.findByPk(aircraft_id);
  if (!aircraft) {
    throw new Error("Aircraft not found");
  }

  // 2. Prevent assigning aircraft under maintenance
  if (aircraft.maintenance_status === "Under Maintenance") {
    throw new Error("Aircraft is under maintenance");
  }

  // 3. Check route exists
  const route = await Route.findByPk(route_id);
  if (!route) {
    throw new Error("Route not found");
  }

  // 4. Prevent same aircraft double booking at same time
  const existingFlight = await Flight.findOne({
    where: {
      aircraft_id,
      departure_time,
    },
  });

  if (existingFlight) {
    throw new Error("Aircraft already assigned to another flight at this time");
  }

  return await Flight.create(data);
};

// Get All Flights
exports.getAllFlights = async () => {
  return await Flight.findAll({
    include: [
      { model: Aircraft },
      { model: Route },
    ],
  });
};

// Get Flight By ID
exports.getFlightById = async (id) => {
  const flight = await Flight.findByPk(id, {
    include: [Aircraft, Route],
  });

  if (!flight) {
    throw new Error("Flight not found");
  }

  return flight;
};

// Update Flight
exports.updateFlight = async (id, data) => {
  const flight = await Flight.findByPk(id);
  if (!flight) {
    throw new Error("Flight not found");
  }

  await flight.update(data);
  return flight;
};

// Update Flight Status
exports.updateFlightStatus = async (id, status) => {
  const flight = await Flight.findByPk(id);
  if (!flight) {
    throw new Error("Flight not found");
  }

  await flight.update({ status });
  return flight;
};

// Delete Flight
exports.deleteFlight = async (id) => {
  const flight = await Flight.findByPk(id);
  if (!flight) {
    throw new Error("Flight not found");
  }

  await flight.destroy();
  return true;
};
✅ 2️⃣ Flight Controller
📁 controllers/flight.controller.js
JavaScript
Copy code
const flightService = require("../services/flight.service");

// Create Flight
exports.createFlight = async (req, res) => {
  try {
    const flight = await flightService.createFlight(req.body);
    res.status(201).json(flight);
  } catch (error) {
    res.status(400).json({ message: error.message });
  }
};

// Get All Flights
exports.getAllFlights = async (req, res) => {
  try {
    const flights = await flightService.getAllFlights();
    res.status(200).json(flights);
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
};

// Get Flight By ID
exports.getFlightById = async (req, res) => {
  try {
    const flight = await flightService.getFlightById(req.params.id);
    res.status(200).json(flight);
  } catch (error) {
    res.status(404).json({ message: error.message });
  }
};

// Update Flight
exports.updateFlight = async (req, res) => {
  try {
    const flight = await flightService.updateFlight(req.params.id, req.body);
    res.status(200).json(flight);
  } catch (error) {
    res.status(404).json({ message: error.message });
  }
};

// Update Flight Status
exports.updateFlightStatus = async (req, res) => {
  try {
    const flight = await flightService.updateFlightStatus(
      req.params.id,
      req.body.status
    );
    res.status(200).json(flight);
  } catch (error) {
    res.status(404).json({ message: error.message });
  }
};

// Delete Flight
exports.deleteFlight = async (req, res) => {
  try {
    await flightService.deleteFlight(req.params.id);
    res.status(200).json({ message: "Flight deleted successfully" });
  } catch (error) {
    res.status(404).json({ message: error.message });
  }
};
