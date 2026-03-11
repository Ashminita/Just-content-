import {
  Flight,
  OperationalEvent,
  FlightPerformance,
  AnalyticsSummary
} from "@package/shared-database";

export class AnalyticsRepository {

  async generateAnalyticsForDate(date: string) {

    const flights = await Flight.findAll({
      where: { flight_date: date }
    });

    for (const flight of flights) {

      const operationalEvents = await OperationalEvent.findAll({
        where: { flight_id: flight.id }
      });

      const performance = await FlightPerformance.findOne({
        where: { flight_id: flight.id }
      });

      const totalFlights = 1;

      const delayedFlights = operationalEvents.filter(
        e => e.delay_minutes > 0
      ).length;

      const avgDelay =
        operationalEvents.length > 0
          ? operationalEvents.reduce((sum, e) => sum + e.delay_minutes, 0) /
            operationalEvents.length
          : 0;

      const avgFlightTime = performance?.flight_time_minutes ?? 0;

      const avgLoadFactor = performance?.load_factor_pct ?? 0;

      const avgFuelEfficiency = performance?.fuel_efficiency_kg_per_km ?? 0;

      await AnalyticsSummary.create({
        date: flight.flight_date,
        origin_airport: flight.origin_airport,
        destination_airport: flight.destination_airport,
        aircraft_id: flight.aircraft_id,
        total_flights: totalFlights,
        delayed_flights: delayedFlights,
        avg_delay_minutes: avgDelay,
        avg_flight_time_minutes: avgFlightTime,
        avg_load_factor_pct: avgLoadFactor,
        avg_fuel_efficiency: avgFuelEfficiency
      });

    }

  }

}

export const analyticsRepository = new AnalyticsRepository();


import { analyticsRepository } from "../repositories/analytics.repository";

class AnalyticsService {

  async generateAnalytics() {

    const today = new Date().toISOString().split("T")[0];

    await analyticsRepository.generateAnalyticsForDate(today);

  }

}

export const analyticsService = new AnalyticsService();


import cron from "node-cron";
import { analyticsService } from "../services/analytics.service";

export function startAnalyticsCron() {

  cron.schedule("0 1 * * *", async () => {

    try {
      await analyticsService.generateAnalytics();
      console.log("Analytics generated");

    } catch (error) {
      console.error("Analytics cron error", error);
    }

  });

}
