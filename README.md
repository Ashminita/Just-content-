import { Flight } from "../models/flight";
import { FlightCrew } from "../models/flight-crew";
import { Crew } from "../models/crew";
import axios from "axios";
import { publishEvent } from "../rabbitmq/publisher";

const AIRCRAFT_SERVICE = "http://localhost:3002/api/aircraft";

export const createFlightService = async (data:any) => {

    // Verify aircraft exists
    const aircraft = await axios.get(`${AIRCRAFT_SERVICE}/${data.aircraft_id}`);

    if(!aircraft.data)
        throw new Error("Aircraft not found");


    const flight = await Flight.create(data);


    await publishEvent("flight.created",{
        flightId:flight.id
    })

    return flight;
};



export const getFlightsService = async () => {

    return Flight.findAll();

};


export const getFlightByIdService = async(id:string)=>{

    return Flight.findByPk(id);

};



export const updateFlightStatusService = async(
    id:string,
    status:string,
    delay_reason?:string,
    delay_minutes?:number
)=>{


    const flight = await Flight.findByPk(id);

    if(!flight)
        throw new Error("Flight not found");


    await flight.update({
        status,
        delay_reason,
        delay_minutes
    })


    await publishEvent("flight.status.updated",{
        flightId:id,
        status
    })


    return flight;
};



export const assignCrewService = async(
    flight_id:string,
    crew_id:string
)=>{


    const flight = await Flight.findByPk(flight_id);

    if(!flight)
        throw new Error("Flight not found");


    const crew = await Crew.findByPk(crew_id);

    if(!crew)
        throw new Error("Crew not found");


    return FlightCrew.create({
        flight_id,
        crew_id
    });

};
import { Request,Response } from "express";
import * as service from "../services/flight-service";



export const createFlight = async(
    req:Request,
    res:Response
)=>{

    try{

        const flight = await service.createFlightService(
            req.body
        )

        res.status(201).json(flight)

    }
    catch(err:any){

        res.status(500).json({
            message:err.message
        })

    }

};



export const getFlights = async(
    req:Request,
    res:Response
)=>{


    const flights = await service.getFlightsService();

    res.json(flights)

};



export const getFlightById = async(
    req:Request,
    res:Response
)=>{

    const flight = await service.getFlightByIdService(
        req.params.id
    )

    res.json(flight)

};



export const updateFlightStatus = async(
    req:Request,
    res:Response
)=>{


    const flight = await service.updateFlightStatusService(
        req.params.id,
        req.body.status,
        req.body.delay_reason,
        req.body.delay_minutes
    )

    res.json(flight)

};



export const assignCrew = async(
    req:Request,
    res:Response
)=>{


    const record = await service.assignCrewService(
        req.body.flight_id,
        req.body.crew_id
    )

    res.json(record)

};
import express from "express";
import * as controller from "../controllers/flight-controller";

const router = express.Router();



router.post(
    "/",
    controller.createFlight
);


router.get(
    "/",
    controller.getFlights
);


router.get(
    "/:id",
    controller.getFlightById
);


router.patch(
    "/:id/status",
    controller.updateFlightStatus
);


router.post(
    "/assign-crew",
    controller.assignCrew
);



export default router;

import { Crew } from "../models/crew";

export const createCrewService = async (data:any) => {
  return await Crew.create(data);
};

export const getAllCrewService = async () => {
  return await Crew.findAll();
};

export const getCrewByIdService = async (id:string) => {
  return await Crew.findByPk(id);
};

export const updateCrewService = async (id:string,data:any) => {
  await Crew.update(data,{where:{id}});
  return await Crew.findByPk(id);
};

export const deleteCrewService = async (id:string) => {
  return await Crew.destroy({where:{id}});
};
