<table>

<tr>
<th>Flight</th>
<th>Route</th>
<th>Status</th>
<th>Aircraft</th>
</tr>

{flights.map(f=>(
<tr key={f.id}>

<td>{f.flight_number}</td>

<td>
{f.departure_airport} → {f.arrival_airport}
</td>

<td>{f.status}</td>

<td>
{f.Aircraft?.registration_number}
</td>

</tr>
))}

</table>
useEffect(()=>{

fetch("http://localhost:3001/api/dashboard",{

credentials:"include"

})
.then(res=>res.json())
.then(setFlights)

},[])

[
{
"id":"f123",

"flight_number":"AI101",

"departure_airport":"DEL",

"arrival_airport":"BOM",

"status":"Delayed",

"Aircraft":{
"registration_number":"VT123",
"model":"A320"
}
}
]

Flight.belongsTo(Aircraft,{
 foreignKey:"aircraft_id"
});

Aircraft.hasMany(Flight,{
 foreignKey:"aircraft_id"
});

import {
Flight,
Aircraft
} from "@repo/shared-database";


export const getDashboard = async()=>{

 const flights = await Flight.findAll({

 include:[
 {
 model:Aircraft,
 attributes:["registration_number","model"]
 }
 ],

 order:[
 ["departure_date","DESC"]
 ]

 });

 return flights;

};

import { Request,Response,NextFunction } from "express";
import * as dashboardService from "../services/dashboard-service";


export const getDashboard = async(
 req:Request,
 res:Response,
 next:NextFunction
)=>{

 try{

 const data = await dashboardService.getDashboard();

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
