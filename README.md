import { DataTypes, Model, Optional } from "sequelize";
import { sequelize } from "../config/database";

interface FlightPerformanceAttributes {
  id: string;
  flight_id: string;
  fuel_used_kg: number;
  distance_km: number;
  passengers_boarded: number;
  seat_capacity: number;

  fuel_efficiency: number;
  load_factor_pct: number;
  co2_emissions_kg: number;

  created_at?: Date;
  updated_at?: Date;
}

interface FlightPerformanceCreationAttributes
  extends Optional<
    FlightPerformanceAttributes,
    "id" | "fuel_efficiency" | "load_factor_pct" | "co2_emissions_kg"
  > {}

export class FlightPerformance
  extends Model<
    FlightPerformanceAttributes,
    FlightPerformanceCreationAttributes
  >
  implements FlightPerformanceAttributes
{
  public id!: string;
  public flight_id!: string;
  public fuel_used_kg!: number;
  public distance_km!: number;
  public passengers_boarded!: number;
  public seat_capacity!: number;

  public fuel_efficiency!: number;
  public load_factor_pct!: number;
  public co2_emissions_kg!: number;

  public readonly created_at!: Date;
  public readonly updated_at!: Date;
}

FlightPerformance.init(
  {
    id: {
      type: DataTypes.UUID,
      defaultValue: DataTypes.UUIDV4,
      primaryKey: true,
    },
    flight_id: {
      type: DataTypes.UUID,
      allowNull: false,
      unique: true, // One performance per flight
    },
    fuel_used_kg: {
      type: DataTypes.FLOAT,
      allowNull: false,
    },
    distance_km: {
      type: DataTypes.FLOAT,
      allowNull: false,
    },
    passengers_boarded: {
      type: DataTypes.INTEGER,
      allowNull: false,
    },
    seat_capacity: {
      type: DataTypes.INTEGER,
      allowNull: false,
    },
    fuel_efficiency: {
      type: DataTypes.FLOAT,
      allowNull: false,
    },
    load_factor_pct: {
      type: DataTypes.FLOAT,
      allowNull: false,
    },
    co2_emissions_kg: {
      type: DataTypes.FLOAT,
      allowNull: false,
    },
  },
  {
    sequelize,
    tableName: "flight_performance",
    underscored: true,
  }
);

import { z } from "zod";

export const createFlightPerformanceSchema = z.object({
  flight_id: z.string().uuid(),

  fuel_used_kg: z.number().positive(),
  distance_km: z.number().positive(),

  passengers_boarded: z.number().int().min(0),
  seat_capacity: z.number().int().positive(),
});

import { FlightPerformance } from "@shared/models/flightPerformance.model";
import { Flight } from "@shared/models/flight.model";

export class FlightPerformanceRepository {
  async findFlightById(id: string) {
    return Flight.findByPk(id);
  }

  async findByFlightId(flightId: string) {
    return FlightPerformance.findOne({ where: { flight_id: flightId } });
  }

  async create(data: any) {
    return FlightPerformance.create(data);
  }

  async findAll() {
    return FlightPerformance.findAll({
      order: [["created_at", "DESC"]],
    });
  }
}

import { FlightPerformanceRepository } from "../repositories/flightPerformance.repository";
import { ApiError } from "@shared/utils/apiError";

export class FlightPerformanceService {
  private repository = new FlightPerformanceRepository();

  async createPerformance(data: {
    flight_id: string;
    fuel_used_kg: number;
    distance_km: number;
    passengers_boarded: number;
    seat_capacity: number;
  }) {
    const flight = await this.repository.findFlightById(data.flight_id);

    if (!flight) {
      throw new ApiError(404, "Flight not found");
    }

    if (flight.status !== "completed") {
      throw new ApiError(
        400,
        "Performance can only be recorded for completed flights"
      );
    }

    const existing = await this.repository.findByFlightId(data.flight_id);

    if (existing) {
      throw new ApiError(
        409,
        "Performance record already exists for this flight"
      );
    }

    // AUTO CALCULATIONS

    const fuel_efficiency =
      data.fuel_used_kg / data.distance_km;

    const load_factor_pct =
      (data.passengers_boarded / data.seat_capacity) * 100;

    const co2_emissions_kg =
      data.fuel_used_kg * 3.16;

    return this.repository.create({
      ...data,
      fuel_efficiency: Number(fuel_efficiency.toFixed(2)),
      load_factor_pct: Number(load_factor_pct.toFixed(2)),
      co2_emissions_kg: Number(co2_emissions_kg.toFixed(2)),
    });
  }

  async getAllPerformance() {
    return this.repository.findAll();
  }
}

import { Request, Response, NextFunction } from "express";
import { FlightPerformanceService } from "../services/flightPerformance.service";
import { createFlightPerformanceSchema } from "../validations/flightPerformance.validation";

const service = new FlightPerformanceService();

export class FlightPerformanceController {
  async create(req: Request, res: Response, next: NextFunction) {
    try {
      const validated = createFlightPerformanceSchema.parse(req.body);

      const performance = await service.createPerformance(validated);

      return res.status(201).json({
        success: true,
        message: "Flight performance recorded successfully",
        data: performance,
      });
    } catch (error) {
      next(error);
    }
  }

  async getAll(req: Request, res: Response, next: NextFunction) {
    try {
      const data = await service.getAllPerformance();

      return res.status(200).json({
        success: true,
        data,
      });
    } catch (error) {
      next(error);
    }
  }
}

import { Router } from "express";
import { FlightPerformanceController } from "../controllers/flightPerformance.controller";
import { authenticate } from "@shared/middlewares/authenticate";
import { authorize } from "@shared/middlewares/authorize";

const router = Router();
const controller = new FlightPerformanceController();

router.use(authenticate);

router.post(
  "/",
  authorize(["admin", "operation"]),
  controller.create.bind(controller)
);

router.get(
  "/",
  authorize(["admin", "analyst", "manager"]),
  controller.getAll.bind(controller)
);

export default router;

✅ PART 1 — PAGINATION + FILTERING (Flight Performance)
📁 Update Validation
validations/performanceQuery.validation.ts
Ts
Copy code
import { z } from "zod";

export const performanceQuerySchema = z.object({
  page: z.string().optional(),
  limit: z.string().optional(),
  startDate: z.string().optional(),
  endDate: z.string().optional(),
});
📁 Update Repository
repositories/flightPerformance.repository.ts
Ts
Copy code
import { Op } from "sequelize";
import { FlightPerformance } from "@shared/models/flightPerformance.model";

export class FlightPerformanceRepository {

  async findAllWithPagination({
    page,
    limit,
    startDate,
    endDate,
  }: {
    page: number;
    limit: number;
    startDate?: string;
    endDate?: string;
  }) {

    const offset = (page - 1) * limit;

    const whereClause: any = {};

    if (startDate && endDate) {
      whereClause.created_at = {
        [Op.between]: [new Date(startDate), new Date(endDate)],
      };
    }

    const { rows, count } = await FlightPerformance.findAndCountAll({
      where: whereClause,
      offset,
      limit,
      order: [["created_at", "DESC"]],
    });

    return { rows, count };
  }
}
📁 Update Service
services/flightPerformance.service.ts
Ts
Copy code
async getAllPerformance(query: any) {

  const page = Number(query.page) || 1;
  const limit = Number(query.limit) || 10;

  const { rows, count } =
    await this.repository.findAllWithPagination({
      page,
      limit,
      startDate: query.startDate,
      endDate: query.endDate,
    });

  return {
    data: rows,
    pagination: {
      totalRecords: count,
      totalPages: Math.ceil(count / limit),
      currentPage: page,
      pageSize: limit,
    },
  };
}
📁 Update Controller
Ts
Copy code
import { performanceQuerySchema } from "../validations/performanceQuery.validation";

async getAll(req: Request, res: Response, next: NextFunction) {
  try {
    const validatedQuery = performanceQuerySchema.parse(req.query);

    const result = await service.getAllPerformance(validatedQuery);

    return res.status(200).json({
      success: true,
      ...result,
    });
  } catch (error) {
    next(error);
  }
}
✅ PART 2 — PAGINATION FOR OPERATIONAL EVENTS
📁 Repository Update
repositories/operationalEvent.repository.ts
Ts
Copy code
async findEventsByFlightWithPagination(
  flightId: string,
  page: number,
  limit: number
) {
  const offset = (page - 1) * limit;

  const { rows, count } =
    await OperationalEvent.findAndCountAll({
      where: { flight_id: flightId },
      offset,
      limit,
      order: [["created_at", "DESC"]],
    });

  return { rows, count };
}
📁 Service Update
Ts
Copy code
async getEventsByFlight(flightId: string, query: any) {

  const page = Number(query.page) || 1;
  const limit = Number(query.limit) || 10;

  const { rows, count } =
    await this.repository.findEventsByFlightWithPagination(
      flightId,
      page,
      limit
    );

  return {
    data: rows,
    pagination: {
      totalRecords: count,
      totalPages: Math.ceil(count / limit),
      currentPage: page,
      pageSize: limit,
    },
  };
}


