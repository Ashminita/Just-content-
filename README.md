
};


 res.json(data);

 }
 catch(err){

 next(err);

 }

};

import { Router } from "express";
import { getDashboard } from "../controllers/dashboard-controller";

import { authenticate } from "@repo/shared-utils/authMiddleware";
import { authorize } from "@repo/shared-utils/rbacMiddleware";

const router = Router();

router.get(
"/",
authenticate,
authorize("Manager","Operations"),
getDashboard
);

export default router;

import {
FlightEvent,
Flight
} from "@repo/shared-database";


export const createEvent = async(data:any)=>{

 const flight = await Flight.findByPk(data.flight_id);

 if(!flight)
 throw new Error("Flight not found");


 return FlightEvent.create(data);

};



export const getEvents = async(flightId:string)=>{

 return FlightEvent.findAll({

 where:{
 flight_id:flightId
 }

 });

};

import { Request,Response,NextFunction } from "express";
import * as eventService from "../services/flight-event-service";


export const createEvent = async(
 req:Request,
 res:Response,
 next:NextFunction
)=>{
 try{

 const event = await eventService.createEvent(req.body);

 res.status(201).json(event);

 }catch(err){
 next(err);
 }
};



export const getFlightEvents = async(
 req:Request,
 res:Response,
 next:NextFunction
)=>{
 try{

 const events = await eventService.getEvents(
 req.params.flightId
 );

 res.json(events);

 }catch(err){
 next(err);
 }
};

import { Router } from "express";

import {
createEvent,
getFlightEvents
} from "../controllers/flight-event-controller";

import { authenticate } from "@repo/shared-utils/authMiddleware";
import { authorize } from "@repo/shared-utils/rbacMiddleware";


const router = Router();


router.post(
"/",
authenticate,
authorize("Operations"),
createEvent
);


router.get(
"/:flightId",
authenticate,
authorize("Manager","Operations"),
getFlightEvents
);


export default router;

import { Crew } from "@repo/shared-database";


export const createCrew = async(data:any)=>{

 return Crew.create(data);

};


export const getCrews = async()=>{

 return Crew.findAll();

};


export const getCrewById = async(id:string)=>{

 const crew = await Crew.findByPk(id);

 if(!crew)
 throw new Error("Crew not found");

 return crew;

};


export const updateCrew = async(id:string,data:any)=>{

 const crew = await Crew.findByPk(id);

 if(!crew)
 throw new Error("Crew not found");

 return crew.update(data);

};


export const deleteCrew = async(id:string)=>{

 const crew = await Crew.findByPk(id);

 if(!crew)
 throw new Error("Crew not found");

 await crew.destroy();

};

import { Request,Response,NextFunction } from "express";
import * as crewService from "../services/crew-service";

interface IdParams{
 id:string
}


export const createCrew = async(
 req:Request,
 res:Response,
 next:NextFunction
)=>{
 try{

 const crew = await crewService.createCrew(req.body);

 res.status(201).json(crew);

 }catch(err){
 next(err);
 }
};



export const getCrews = async(
 req:Request,
 res:Response,
 next:NextFunction
)=>{
 try{

 const crews = await crewService.getCrews();

 res.json(crews);

 }catch(err){
 next(err);
 }
};



export const getCrewById = async(
 req:Request<IdParams>,
 res:Response,
 next:NextFunction
)=>{
 try{

 const crew = await crewService.getCrewById(req.params.id);

 res.json(crew);

 }catch(err){
 next(err);
 }
};



export const updateCrew = async(
 req:Request<IdParams>,
 res:Response,
 next:NextFunction
)=>{
 try{

 const crew = await crewService.updateCrew(
 req.params.id,
 req.body
 );

 res.json(crew);

 }catch(err){
 next(err);
 }
};



export const deleteCrew = async(
 req:Request<IdParams>,
 res:Response,
 next:NextFunction
)=>{
 try{

 await crewService.deleteCrew(req.params.id);

 res.json({
 message:"Crew deleted"
 });

 }catch(err){
 next(err);
 }
};

import { Router } from "express";
import {
createCrew,
getCrews,
getCrewById,
updateCrew,
deleteCrew
} from "../controllers/crew-controller";

import { authenticate } from "@repo/shared-utils/authMiddleware";
import { authorize } from "@repo/shared-utils/rbacMiddleware";

const router = Router();

router.post(
"/",
authenticate,
authorize("Operations"),
createCrew
);

router.get(
"/",
authenticate,
authorize("Manager","Operations"),
getCrews
);

router.get(
"/:id",
authenticate,
authorize("Manager","Operations"),
getCrewById
);

router.put(
"/:id",
authenticate,
authorize("Operations"),
updateCrew
);

router.delete(
"/:id",
authenticate,
authorize("Manager"),
deleteCrew
);

export default router;

❤️😊





async createDelayCategory(data) {
  // 1. Validate required fields
  if (!data.code || !data.name)
    throw new Error("Code and Name are required");

  // 2. Check uniqueness
  const exists = await repo.findByCode(data.code);
  if (exists)
    throw new Error("Delay category code already exists");

  // 3. Create category
  return await repo.create(data);
}


async updateDelayCategory(id, data) {
  const category = await repo.findById(id);
  if (!category)
    throw new Error("Category not found");

  if (data.code && data.code !== category.code)
    throw new Error("Code cannot be changed");

  return await repo.update(id, data);
}

async deleteDelayCategory(id) {
  const used = await repo.isUsedInOperationalEvents(id);
  if (used)
    throw new Error("Cannot delete category in use");

  return await repo.delete(id);
}


async createOperationalEvent(flightId, data, userId) {

  // 1. Check flight exists
  const flight = await repo.findFlightById(flightId);
  if (!flight)
    throw new Error("Flight not found");

  // 2. Validate event type
  const validTypes = [
    "delay",
    "diversion",
    "cancellation",
    "equipment_change",
    "gate_change",
    "crew_change",
    "medical",
    "security"
  ];

  if (!validTypes.includes(data.event_type))
    throw new Error("Invalid event type");

  // 3. Delay-specific validation
  if (data.event_type === "delay") {
    if (!data.delay_category_id)
      throw new Error("Delay category required");

    if (!data.delay_minutes || data.delay_minutes <= 0)
      throw new Error("Delay minutes must be positive");

    // Validate delay category exists
    const category = await delayCategoryRepo.findById(data.delay_category_id);
    if (!category)
      throw new Error("Invalid delay category");
  }

  // 4. Validate event_time
  if (!data.event_time)
    throw new Error("Event time required");

  // 5. Create event
  return await repo.create({
    ...data,
    flight_id: flightId,
    reported_by: userId
  });
}


async updateEvent(flightId, eventId, data) {
  const event = await repo.findById(eventId);
  if (!event)
    throw new Error("Event not found");

  if (event.resolved_at)
    throw new Error("Cannot update resolved event");

  return await repo.update(eventId, data);
}


async resolveEvent(eventId) {
  const event = await repo.findById(eventId);
  if (!event)
    throw new Error("Event not found");

  if (event.resolved_at)
    throw new Error("Event already resolved");

  return await repo.resolve(eventId, new Date());
}


