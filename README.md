# Silver-Coast-Nomads
import React, { useState, useEffect, useCallback, useRef } from 'react';
import { initializeApp } from 'firebase/app';
import { 
    getAuth, 
    signInAnonymously, 
    signInWithCustomToken, 
    onAuthStateChanged 
} from 'firebase/auth';
import { 
    getFirestore, 
    collection, 
    query, 
    limit, 
    onSnapshot, 
    addDoc, 
    serverTimestamp,
} from 'firebase/firestore';
import { setLogLevel } from 'firebase/firestore'; 

// Firestore Debugging einschalten
setLogLevel('Debug');

// Globale Variablen aus der Canvas-Umgebung (MANDATORY)
const firebaseConfig = typeof __firebase_config !== 'undefined' ? JSON.parse(__firebase_config) : {};
// FIX: Korrekte Zuweisung des Tokens
const initialAuthToken = typeof __initial_auth_token !== 'undefined' ? __initial_auth_token : null;
const appId = typeof __app_id !== 'undefined' ? __app_id : 'default-app-id';

// Hilfsfunktion zur Berechnung des aktuellen Datumsbereichs der Woche
const getWeeklyDateRange = () => {
    const now = new Date();
    const dayOfWeek = now.getDay(); // 0 = Sonntag, 1 = Montag, ..., 6 = Samstag
    
    // Korrekte Berechnung des Montags (Montag=1)
    const diffToMonday = now.getDate() - dayOfWeek + (dayOfWeek === 0 ? -6 : 1); 
    
    const monday = new Date(now.setDate(diffToMonday));
    // Wir müssen die Zeit neu setzen, um die Datumsberechnung nicht zu verfälschen
    const sunday = new Date(monday.getTime());
    sunday.setDate(sunday.getDate() + 6);

    // Formatierung: Numerisches Datum mit Punkt, ohne Jahr
    const format = (date) => {
        const day = date.getDate();
        const month = date.getMonth() + 1; // Monate sind 0-basiert
        return `${day}.${month}.`;
    };

    return `${format(monday)} – ${format(sunday)}`;
};

// NEU: Hilfsfunktion zur Umrechnung der Windrichtung (Grad) in Surf-Termini
const getWindDirection = (deg) => {
    // Peniche/Baleal hat hauptsächlich West- und Nordwest-Ausrichtung
    let direction = '';
    
    if (deg >= 337.5 || deg < 22.5) direction = 'Nord (N)';
    else if (deg >= 22.5 && deg < 67.5) direction = 'Nordost (NO)';
    else if (deg >= 67.5 && deg < 112.5) direction = 'Ost (O)';
    else if (deg >= 112.5 && deg < 157.5) direction = 'Südost (SO)';
    else if (deg >= 157.5 && deg < 202.5) direction = 'Süd (S)';
    else if (deg >= 202.5 && deg < 247.5) direction = 'Südwest (SW)';
    else if (deg >= 247.5 && deg < 292.5) direction = 'West (W)';
    else if (deg >= 292.5 && deg < 337.5) direction = 'Nordwest (NW)';

    // Offshore (optimal): Ost bis Südost (O, NO, SO)
    if ((deg >= 67.5 && deg < 157.5) || (deg >= 22.5 && deg < 67.5)) {
        return { display: `${direction} (Offshore 💨)`, color: 'text-green-600' };
    }
    // Onshore (schlecht): West bis Nordwest (W, NW)
    if ((deg >= 247.5 && deg < 337.5)) {
        return { display: `${direction} (Onshore 🌬️)`, color: 'text-red-600' };
    }
    // Cross-Shore / Side Shore (neutral): Nord / Süd
    return { display: `${direction} (Neutral ↔️)`, color: 'text-yellow-600' };
};


// SPOT-DATENSTRUKTUR (Statisch) - Surf-Spots, Coworking, Cafés und Bars
const LOCATIONS = [
    // --- SURF SPOTS ---
    { id: 1, name: "Cantinho da Baía", type: "Surf-Spot", zone: "Peniche (Nordbucht)", level: "⭐️ Beginner", detail: "Sehr soft, sanfte Wellen, perfekte Surf-Schul-Bedingungen. Im Süden etwas größer, aber max. Beginner+/Low-Intermediate.", bestWind: "Ost (Offshore)", bestSwell: "Nordwest", rating: 4.8 },
    { id: 2, name: "Meio da Baía", type: "Surf-Spot", zone: "Mid-Bay", level: "⭐️⭐️ Intermediate", detail: "Mehr Swell, schneller und steiler als Cantinho; teils kleine Barrels. Für Intermediates an moderaten Tagen.", bestWind: "Ost (Offshore)", bestSwell: "Nordwest", rating: 4.1 },
    { id: 3, name: "Prainha", type: "Surf-Spot", zone: "Baleal Norte", level: "⭐️ Beginner", detail: "Sehr geschützter Beachbreak, kleine sanfte Wellen – ideal für erste Sessions.", bestWind: "Nord/Nordost (Offshore)", bestSwell: "Nordwest", rating: 4.0 },
    { id: 4, name: "Gigi Beach", type: "Surf-Spot", zone: "Baleal", level: "⭐️ Beginner → ⭐️⭐️ Intermediate", detail: "Ruhiger Strandabschnitt; gut für Longboards. Für alle Levels bei kleinen Swells, aber anspruchsvoller bei größerem Swell.", bestWind: "Ost (Offshore)", bestSwell: "West / Nordwest", rating: 4.3 },
    { id: 5, name: "Lagide", type: "Surf-Spot", zone: "Baleal Reef", level: "⭐️⭐️ Intermediate", detail: "Linker Riff-Break. Sehr konsistent und bietet gute Wellen, wenn Supeertubos zu groß wird. Bei Low Tide flach – nichts für komplette Anfänger.", bestWind: "Ost / Südost (Offshore)", bestSwell: "Nordwest", rating: 4.5 },
    { id: 6, name: "Praia da Almagreira", type: "Surf-Spot", zone: "Baleal", level: "⭐️⭐️ Intermediate → ⭐️⭐️⭐️ Advanced", detail: "Offenes, kraftvolles Beachbreak. Uncrowded aber heftig bei Swell → für erfahrenere Surfer.", bestWind: "Ost", bestSwell: "West / Nordwest", rating: 4.6 },
    { id: 7, name: "Pico da Mota", type: "Surf-Spot", zone: "Belgas", level: "⭐️ Beginner → ⭐️⭐️⭐️ Advanced", detail: "„Alle Level“ bei kleinen Tagen; bei größerem Swell sehr stark & hohl → dann nur für fortgeschrittene/experts.", bestWind: "Ost / Südost", bestSwell: "Nordwest", rating: 4.7 },
    { id: 8, name: "Pedras Muitas", type: "Surf-Spot", zone: "Peniche", level: "⭐️ Beginner → ⭐️⭐️ Intermediate", detail: "Mix aus Sand/Rock, wenig Leute, spaßige Peaks in moderatem Swell – gut geeignet für viele Level.", bestWind: "Ost / Nordost", bestSwell: "West", rating: 4.0 },
    { id: 9, name: "Praia da Gâmboa", type: "Surf-Spot", zone: "Peniche", level: "⭐️ Beginner", detail: "Mellow, sanfte Beachbreak-Wellen. Oft flach wenn es woanders groß ist. Perfekt für Surf-Schulen & erste Sessions.", bestWind: "Ost / Nordost", bestSwell: "Nordwest", rating: 3.5 },
    { id: 10, name: "Praia do Cerro", type: "Surf-Spot", zone: "Peniche", level: "⭐️ Beginner → ⭐️⭐️ Intermediate", detail: "Leichte Peaks, easy paddle-out. Kleine Felszone → Awareness nötig. Ideal für frühe Lern-Phase und Intermediate an moderaten Tagen.", bestWind: "Südost", bestSwell: "Nordwest", rating: 3.9 },
    { id: 11, name: "Supertubos (Medão)", type: "Surf-Spot", zone: "Peniche (Süden)", level: "⭐️⭐️⭐️ Advanced → Pro", detail: "Extrem schnelle, hohle Tubes. Europas Pipeline. Bei Größe nur für sehr erfahrene Surfer*innen. Gefährlich bei Low Tide.", bestWind: "Nordost (Offshore)", bestSwell: "West / Südwest", rating: 5.0 },
    { id: 12, name: "Molhe Leste", type: "Surf-Spot", zone: "Peniche (Hafen)", level: "⭐️⭐️ Intermediate → ⭐️⭐️⭐️ Advanced", detail: "Punchy, wedgy Beachbreak am Hafenmole. Technisch aber kontrollierbarer als Supertubos. Starke Strömungen – Intermediate+ aufwärts.", bestWind: "Ost / Nordost", bestSwell: "Südwest / West", rating: 4.5 },
    { id: 13, name: "Consolação (Point)", type: "Surf-Spot", zone: "Consolação", level: "⭐️⭐️⭐️ Advanced", detail: "Langer Right-Hand Reef/Point. Perfekte Walls bei gutem Sand. Flaches Riff → Erfahrung nötig. Booties empfohlen.", bestWind: "Südost", bestSwell: "Nordwest", rating: 4.7 },
    { id: 14, name: "Praia da Consolação", type: "Surf-Spot", zone: "Consolação", level: "⭐️ Beginner → ⭐️⭐️ Intermediate", detail: "Sehr geschützt; kleine, sanfte Wellen. Ideal für Anfänger an kleinen Tagen, aber bei mehr Swell auch für Intermediates gut.", bestWind: "Ost", bestSwell: "Nordwest", rating: 4.0 },
    { id: 15, name: "Papôa Reef", type: "Surf-Spot", zone: "Peniche Spitze", level: "⭐️⭐️⭐️ Advanced → Big Wave", detail: "Massives Reef, große Barrels bei >10 ft. Anspruchsvoll und gefährlich. Primär ein Heavy-Wave-Spot.", bestWind: "Ost", bestSwell: "Nordwest", rating: 4.9 },
    { id: 16, name: "Praia do Portinho da Areia", type: "Surf-Spot", zone: "Peniche", level: "⭐️⭐️⭐️ Advanced", detail: "Selten surfbar, intensiver Left-Hander bei großem NW-Swell. Kurz, kräftig, oft bodyboard-lastig. Hohe technische Ansprüche.", bestWind: "Nordost", bestSwell: "Nordwest", rating: 4.2 },
    { id: 17, name: "Praia d’El Rei", type: "Surf-Spot", zone: "Südlich von Peniche", level: "⭐️⭐️ Intermediate", detail: "Sacht ansteigende Beachbreak-Peaks, Swell-Magnet. Kann schnell zu groß werden – aber ideal für Intermediates an mittelgroßen Tagen.", bestWind: "Ost", bestSwell: "West", rating: 4.1 },
    // --- COWORKING SPACES ---
    { id: 18, name: "RIDE Surf Resort CoWorking", type: "Coworking", zone: "Baleal", level: "Highspeed-WLAN, Pool/Gym", detail: "5 Min zu Fuß vom Strand. Cowork-Lounge & Meetingraum in einem Surf-Resort/Spa. Highspeed-WLAN, viele Plätze & Steckdosen, Zugang zu Café/Bar. Halbtag (€7) oder Ganztag (€12) inkl. Guthaben fürs Restaurant. Extra: Skate-Ramp, Pool, Gym – perfekt zum Abschalten nach der Arbeit.", wifi: "Highspeed", rating: 4.8 },
    { id: 19, name: "PipeDream Hub", type: "Coworking", zone: "Ferrel", level: "800 Mbit/s Internet", detail: "Coliving-/Coworking-Haus in Casais do Baleal, fußläufig zu den Stränden. Glasüberdachtes „PipeHub“-Office mit Poolblick, 800 Mbit/s Internet, ergonomische Stühle, feste Schreibtische, Hängematten. Sehr community-orientiert: Arbeiten & Surfen mit Gleichgesinnten.", wifi: "800 Mbit/s", rating: 4.5 },
    { id: 20, name: "Secret Surf House Office", type: "Coworking", zone: "Ferrel", level: "250 Mbit/s Hybrid-WLAN", detail: "Co-Living-Gästehaus mit hervorragenden Arbeitsbereichen: Gemeinschaftstisch, private Büros, auf Wunsch Arbeitsplatz im Zimmer. 250 Mbit/s Hybrid-WLAN. Beliebt bei Berufstätigen, die Work & Surf kombinieren wollen.", wifi: "250 Mbit/s", rating: 4.6 },
    { id: 21, name: "Swelldesk Coliving", type: "Coworking", zone: "Atouguia da Baleia", level: "~200 Mbit/s WLAN", detail: "Kreative Co-Living-/Cowork-Villa ca. 2 km von Baleal. Moderner, offener Workspace mit ~200 Mbit/s WLAN, großen Tischen und ruhiger Lage. Platz für bis zu 6 Coworker – ideal für Teams oder Freundesgruppen.", wifi: "200 Mbit/s", rating: 4.7 },
    { id: 22, name: "Tribe Cowork (Ferrel)", type: "Coworking", zone: "Ferrel", level: "Schnelle Internetverbindung, 24/7", detail: "Kleiner, freundlicher Community-Coworking-Space im Dorfzentrum. 24/7 Zugang, schnelle Internetverbindung, einige Telefonkabinen, oft Workshops & Skill-Share Sessions.", wifi: "Schnell", rating: 4.3 },
    { id: 23, name: "In My House – Coworking", type: "Coworking", zone: "Baleal", level: "Starkes WLAN, Meerblick (Gäste)", detail: "Surf-Gästehaus am Strand mit eigenem Cowork-Bereich (nur für Gäste). Großer Gemeinschaftstisch, Monitor-Verleih, starkes WLAN und Terrasse mit Meerblick – ideal fürs Arbeiten zwischen Surfsessions.", wifi: "Stark", rating: 4.9 },
    { id: 24, name: "Baleal Surf Camp Office", type: "Coworking", zone: "Baleal Island", level: "Gutes WLAN, Draußen Wellen checken", detail: "Surf-Camp, das in der Off-Season eine kleine Cowork-Lounge anbietet. Schlichter Workspace mit ein paar Schreibtischen, gutem WLAN und gratis Kaffee/Snacks. Highlight: Draußen direkt Wellen checken.", wifi: "Gut", rating: 3.8 },
    { id: 25, name: "Malacuna Peniche", type: "Coworking", zone: "Baleal / Peniche Grenze", level: "Sehr schnelles Internet, viele Events", detail: "Bekanntes Nomad-Hub mit Hostel + Coworking. Große offene Arbeitsbereiche, private Kabinen, Bibliothek, Café vor Ort und viele Events. Sehr schnelles Internet. Ehemals Selina Peniche, jetzt modernisiert („Malacuna“).", wifi: "Sehr schnell", rating: 4.7 },
    { id: 26, name: "Largo Space (Peniche)", type: "Coworking", zone: "Peniche", level: "Hot-Desks ab €8/Tag, 24/7 Zugang", detail: "Kreativer Cowork-Hub im Zentrum (Largo dos Galeões). Hot-Desks ab ca. €8/Tag oder €80/Monat. 24/7 Zugang. Mischung aus Freelancern, Gründern & Kreativen. Büroähnliche Umgebung – perfekt, wenn du Ruhe vom Strand suchst.", wifi: "Schnell", rating: 4.4 },
    { id: 27, name: "Silvercoast Virtual Office", type: "Coworking", zone: "Peniche", level: "Fokus auf ruhigem Arbeiten", detail: "Kleines Business-Center mit Cowork-Plätzen & virtuellem Büroservice. Fokus auf ruhigem Arbeiten statt Community. Ideal bei Bedarf nach Geschäftsadresse oder Meetingraum.", wifi: "Verfügbar", rating: 3.5 },
    // --- CAFÉS (Alle auf type: "Cafés" umgestellt) ---
    { id: 28, name: "Pingo Doce Café", type: "Cafés", zone: "Peniche Zentrum", level: "Starkes WLAN", detail: "Typisches portugiesisches Café im Supermarkt. Gut für schnellen Kaffee und kurze Sessions.", wifi: "20 Mbit/s", rating: 3.5 },
    { id: 29, name: "Washed Up Café", type: "Cafés", zone: "Ferrel", level: "Hippes, junges Café", detail: "Hippes, junges Café mit Specialty Coffee, Bowls, Sandwiches und entspanntem Surf-Flair. Beliebt bei Expats & Locals. Outdoor Seating, guter Kaffee, stylische Einrichtung.", wifi: "30 Mbit/s", rating: 4.5 },
    { id: 30, name: "The Green Bowl", type: "Cafés", zone: "Ferrel", level: "Healthy Bowls & Smoothies", detail: "Ideal für gesundes Frühstück und Vormittagsarbeit. Tische drinnen und draußen.", wifi: "50 Mbit/s", rating: 4.7 },
    { id: 31, name: "Maktub", type: "Cafés", zone: "Peniche Hafen", level: "Meerblick & gute Snacks", detail: "Café mit Blick auf den Hafen. Ideal, um Wind und Wetter zu beobachten. Schnelles WLAN, ruhig.", wifi: "80 Mbit/s", rating: 4.9 },
    { id: 32, name: "A Grelha", type: "Cafés", zone: "Consolação", level: "Lokale Atmosphäre", detail: "Authentisches, älteres Café. WLAN ist für kurze Checks in Ordnung, aber nicht für Videocalls geeignet.", wifi: "5 Mbit/s", rating: 3.2 },
    { id: 33, name: "Surfside Terrace", type: "Cafés", zone: "Baleal Beach", level: "Direkt am Strand", detail: "Toller Spot, um die Wellen zu beobachten. Gutes Außen-WLAN. Kann an sonnigen Tagen laut sein.", wifi: "40 Mbit/s", rating: 4.4 },
    { id: 34, name: "Café do Sol", type: "Cafés", zone: "Peniche Zentrum", level: "Gute Kaffeebohnen", detail: "Kleines, verstecktes Café mit Fokus auf hochwertigem Kaffee. Sehr ruhig, ideal zum tiefen Arbeiten.", wifi: "70 Mbit/s", rating: 4.6 },
    { id: 35, name: "Beach House Cafe", type: "Cafés", zone: "Ferrel", level: "Lounge-Musik & bequeme Sofas", detail: "Entspannte Lounge-Atmosphäre, beliebt bei Locals. Guter WLAN-Empfang.", wifi: "60 Mbit/s", rating: 4.2 },
    { id: 36, name: "Pastelaria Peniche", type: "Cafés", zone: "Peniche Zentrum", level: "Traditionelle Pastéis de Nata", detail: "Ein Muss für Süßes. WLAN nur für Notfälle, hier dominiert die lokale Konditorei.", wifi: "10 Mbit/s", rating: 4.0 },
    { id: 37, name: "Nomad Fuel Station", type: "Cafés", zone: "Baleal", level: "Nomad-gerecht, viele Steckdosen", detail: "Eindeutig auf digitale Nomaden ausgerichtet. Etwas höhere Preise, aber garantiert guter Arbeitsplatz.", wifi: "100 Mbit/s", rating: 5.0 },
    { id: 38, name: "Sol Café", type: "Cafés", zone: "Ferrel", level: "Specialty Coffee, vegane Optionen", detail: "Trendiges Specialty-Coffee- und Natural-Wine-Café mit entspanntem Surf-Vibe. AeroPress/V60, vegane Optionen, schnelles WLAN – ideal für digitale Nomaden.", wifi: "Schnell", rating: 4.8 },
    { id: 39, name: "Aloha Café", type: "Cafés", zone: "Ferrel", level: "Boho-Vibe, Healthy Bowls", detail: "Boho-Café mit sehr gutem Kaffee, Healthy Bowls und stabilem WLAN. Surf-Atmosphäre, viele Remote Worker.", wifi: "Stabil", rating: 4.6 },
    { id: 40, name: "Danau Surf Café", type: "Cafés", zone: "Baleal", level: "Strandnah, Meerblick", detail: "Strandnahes Surf-Camp-Café mit starkem Kaffee, Frühstück & Smoothies. Viele Steckdosen, Meerblick – beliebt zum Arbeiten.", wifi: "Stark", rating: 4.3 },
    { id: 41, name: "Café do Mercado", type: "Cafés", zone: "Peniche", level: "Rustikal, lokales Gebäck", detail: "Rustikales Markt-Café mit sehr gutem Gebäck & Kaffee. Authentisch und lokal, WLAN vorhanden.", wifi: "Vorhanden", rating: 3.9 },
    { id: 42, name: "Java House", type: "Cafés", zone: "Peniche", level: "Stylisch, Bäckerei / Bar", detail: "Stylisches Café & Bäckerei mit moderner Atmosphäre. Tagsüber Kaffee & Kuchen, abends Cocktail- und Gin-Bar.", wifi: "Verfügbar", rating: 4.1 },
    { id: 43, name: "Roots Café", type: "Cafés", zone: "Peniche", level: "Vegan/Vegetarisch, ruhige Arbeit", detail: "Veganes/vegetarisches Café mit Specialty-Lattes und ruhiger Arbeitsatmosphäre. Gut geeignet für Freelancer.", wifi: "Ruhig", rating: 4.8 },
    { id: 44, name: "Pequena Baleia", type: "Cafés", zone: "Baleal", level: "Brunch, Sonnenplätze", detail: "Beliebtes Brunch- und Kaffee-Café nahe Baleal Beach. Sonnenplätze, Surf-Interior – gut für Kaffee + Laptop.", wifi: "Gut", rating: 4.5 },
    { id: 45, name: "Mundano Baleal", type: "Cafés", zone: "Baleal Island", level: "Craft Coffee, Meerblick", detail: "Künstlerisches Café/Bar auf Baleal Island. Craft Coffee, Cocktails, Meerblick, entspannte Musik.", wifi: "Verfügbar", rating: 4.7 },
    { id: 46, name: "Surfers Lodge Café", type: "Cafés", zone: "Baleal", level: "Stilvolle Lounge, Nomad-Events", detail: "Stilvolle Lounge mit Espresso, Smoothies & Snacks. WLAN, viel Platz, häufig Nomad-Events.", wifi: "Verlässlich", rating: 4.6 },
    { id: 47, name: "HOWM Café (Malacuna)", type: "Cafés", zone: "Baleal", level: "Nomad-friendly, Specialty Coffee", detail: "Im Selina/Malacuna Hostel. Specialty Coffee, Brunch, Meerblick & verlässliches WLAN – sehr nomad-friendly.", wifi: "Sehr verlässlich", rating: 4.9 },
    { id: 48, name: "Kalifood", type: "Cafés", zone: "Baleal", level: "Gesundes Brunch-Café", detail: "Gesundes Brunch-Café & Restaurant mit Surf-Lifestyle. Frische Smoothie Bowls, Paninis, Kaffee, viele vegetarische Optionen. Perfekt für Post-Surf-Brunch.", wifi: "Stabil", rating: 4.5 },
    { id: 49, name: "Tugos Café", type: "Cafés", zone: "Ferrel", level: "Gemütlich, lokale Stimmung", detail: "Kleines, gemütliches Café/Backery direkt im Dorf. Guter portugiesischer Kaffee, köstliche Gebäckstücke und herzliche lokale Stimmung – ideal für ein schnelles Frühstück.", wifi: "Standard", rating: 4.0 },
    { id: 50, name: "Marquito", type: "Cafés", zone: "Baleal / Ferrel", level: "Traditionell & Günstig", detail: "Traditionelles portugiesisches Café bei Baleal/Ferrel. Bekannt für günstigen Kaffee, Tostas, Pastéis de Nata und authentischen lokalen Vibe – ein klassischer Treffpunkt.", wifi: "Standard", rating: 3.8 },
    // --- BARS ---
    { id: 51, name: "Bar da Praia", type: "Bars", zone: "Baleal Beach", level: "Live Musik & Cocktails", detail: "Kultige Strandbar direkt im Sand. Berühmt für Sonnenuntergänge, DJs und Surf-Party-Vibe. Tagsüber entspannt, abends tanzen.", wifi: "Verfügbar", rating: 4.9 },
    { id: 52, name: "Danau Bar", type: "Bars", zone: "Baleal", level: "Clubartige Strandbar", detail: "Größter Partyspot in Baleal. Clubartige Strandbar, Surf-Backpacker-Crowd, Feiern bis in die frühen Morgenstunden.", wifi: "Schwach", rating: 4.0 },
    { id: 53, name: "Bar do Bruno", type: "Bars", zone: "Baleal", level: "Legendär, DJs/Live-Musik", detail: "Legendäre Surferbar in den Dünen. Ungezwungen, tolle Sonnenuntergänge, oft Live-Musik oder DJs.", wifi: "Vorhanden", rating: 4.6 },
    { id: 54, name: "Bocaxica Surf Bar", type: "Bars", zone: "Baleal", level: "Burger, Drinks & Live-Musik", detail: "Beliebt für Burger, Drinks & Live-Musik. Entspannter Surf-Vibe, moderner Style.", wifi: "Stabil", rating: 4.3 },
    { id: 55, name: "Gauchão da Baleal", type: "Bars", zone: "Baleal", level: "Brasilianische Vibes, Panoramablick", detail: "Brasilianisch inspirierte Bar/Grill. Panoramablick, Sunset-Cocktails, gelegentlich Samba/Bossa Nova.", wifi: "Verfügbar", rating: 4.5 },
    { id: 56, name: "Prainha Bar", type: "Bars", zone: "Baleal Norte / Prainha", level: "Ruhig, Beanbags im Sand", detail: "Ruhige Strandbar/Restaurant. Drinks am Abend auf Beanbags im Sand – perfekt für entspannte Abende.", wifi: "Vorhanden", rating: 4.7 },
    { id: 57, name: "58 Surf Bar", type: "Bars", zone: "Baleal", level: "Craft Beer, Surf-Videos", detail: "Teil des 58 Surf Shops. Tagsüber Café, abends Treffpunkt für Surfer. Craft Beer, Cocktails, Surf-Videos.", wifi: "Stabil", rating: 4.4 },
    { id: 58, name: "The Base", type: "Bars", zone: "Baleal", level: "Social Bar & Community-Spot", detail: "Großer Garten, Events, Open Mic, Ping-Pong – ideal, um andere Surfer & Nomads zu treffen.", wifi: "Gut", rating: 4.6 },
    { id: 59, name: "Berlenga’s", type: "Bars", zone: "Ferrel", level: "Gemütlicher Pub, Sport", detail: "Gemütlicher Pub mit portugiesischem Bier & Snacks. Lokale & Expats mischen sich, oft Sportübertragungen.", wifi: "Vorhanden", rating: 4.0 },
    { id: 60, name: "Ricle Bar", type: "Bars", zone: "Ferrel", level: "Live-Musik, DJ Nights", detail: "Moderner Bar/Club-Mix. Bekannt für Live-Musik, DJ Nights und regelmäßige Events. Beliebt bei der jungen Surf-Szene.", wifi: "Verfügbar", rating: 4.2 },
    { id: 61, name: "Bar O Burro", type: "Bars", zone: "Ferrel", level: "Alternative, rustikal", detail: "Alternative Surf-Bar mit lässigem, etwas rustikalem Ambiente. Gute Drinks, oft spontane Live-Abende.", wifi: "Schwach", rating: 3.7 },
    { id: 62, name: "You Drinks & Food", type: "Bars", zone: "Ferrel", level: "Freundlich, gute Cocktails", detail: "Bar & Snack-Spot mit entspanntem Vibe, freundlicher Atmosphäre und guten Cocktails. Perfekt für Chill-Abende mit Freunden.", wifi: "Stabil", rating: 4.1 },
    { id: 63, name: "Bar do Quebrado", type: "Bars", zone: "Peniche (Nordbucht)", level: "Strandbar, Petiscos", detail: "Lebhafte Strandbar mit lokalen Snacks (Petiscos). Perfekt für Bier oder Ginginha am Meer.", wifi: "Verfügbar", rating: 4.4 },
    { id: 64, name: "Xakra Beach Bar", type: "Bars", zone: "Consolação", level: "Trendig, Meerblick, Tapas", detail: "Trendige Beachbar mit Tapas und grandiosem Meerblick. Sunset-Spot mit DJ Nights.", wifi: "Verfügbar", rating: 4.8 },
    { id: 65, name: "Peniche “Banana” Beach Bar", type: "Bars", zone: "Peniche Beaches", level: "Sommer-Popup, Cocktails", detail: "Sommer-Popup-Bar im Sand. Tropische Cocktails & Party unter den Sternen.", wifi: "Kein WLAN", rating: 4.5 },
    { id: 66, name: "Supertubos Beach Bar", type: "Bars", zone: "Supertubos", level: "Chillig, Musik & Drinks", detail: "Chillige Strandbar. Tagsüber Café, abends Musik & Drinks. Ideal nach der Surfsession.", wifi: "Verfügbar", rating: 4.3 },
    { id: 67, name: "Gamboa Bar", type: "Bars", zone: "Gamboa Beach, Peniche", level: "Fischerei-Flair, Musik", detail: "Mischung aus Fischerei-Ortsflair & Surf-Vibe. Sangria, Bier, manchmal Fado/Rock-Concerts.", wifi: "Verfügbar", rating: 4.1 },
    { id: 68, name: "Bar Java House", type: "Bars", zone: "Peniche", level: "Stylisch, Cocktail-Lounge", detail: "Stylische Café-Bar. Abends Cocktail-Lounge mit Live-Musik und chilligem Ambiente.", wifi: "Stabil", rating: 4.6 },
    { id: 69, name: "Bar Três As", type: "Bars", zone: "Peniche", level: "Lokal, Günstig, Billard", detail: "Klassische lokale Kneipe („Drei Asse“). Günstige Drinks, Billard, Fußball im TV, authentisch & freundlich.", wifi: "Nicht vorhanden", rating: 3.9 },
    { id: 70, name: "Os Americanos", type: "Bars", zone: "Peniche", level: "Unkonventionell, Wein/Bier", detail: "Unkonventionelle Bar mit Americana & Portugal-Mix. Große Bier-/Weinauswahl, gemütlich, ideal für ruhige Abende.", wifi: "Verfügbar", rating: 4.2 },
    { id: 71, name: "Berlenga’s Pub", type: "Bars", zone: "Peniche", level: "Irischer Pub, Live-Musik", detail: "Irischer Pub mit Guinness & Live-Musik. Warmherzige Atmosphäre, beliebt bei Travellern.", wifi: "Stabil", rating: 4.5 },
    { id: 72, name: "2520 Bar", type: "Bars", zone: "Peniche", level: "Modern, DJ & Party-Spot", detail: "Moderne Bar (benannt nach der PLZ). Tagsüber Café, abends DJs & Motto-Partys – junger Partyspot.", wifi: "Verfügbar", rating: 4.0 },
    // --- RESTAURANTS (NEU HINZUGEFÜGT) ---
    { 
        id: 73, 
        name: "Cantina De Ferrel", 
        type: "Restaurants", 
        zone: "Baleal / Ferrel", 
        level: "🌱 Veggie/Vegan Optionen", 
        detail: "Mediterrane / italienische Küche & Pizza, auch vegetarische/vegane Optionen – gute Bewertungen für Essen & Service.",
        wifi: "Gut",
        rating: 4.4 
    },
    { 
        id: 74, 
        name: "Mundano Baleal", 
        type: "Restaurants", 
        zone: "Baleal", 
        level: "🌱 Vegan/Vegetarisch, frische Küche", 
        detail: "Bekannt für vegane/vegetarische Gerichte, frische Küche und gutes Preis-Leistungs-Verhältnis — beliebt bei Nomads.",
        wifi: "Stabil",
        rating: 4.6 
    },
    { 
        id: 75, 
        name: "Taberna Dos Almocreves", 
        type: "Restaurants", 
        zone: "Ferrel", 
        level: "Portugiesische Küche", 
        detail: "Mediterrane/portugiesische Küche, solide Bewertungen — gutes Preis-Leistungs-Verhältnis.",
        wifi: "Vorhanden",
        rating: 4.2 
    },
    { 
        id: 76, 
        name: "O Cantinho Saloio", 
        type: "Restaurants", 
        zone: "Ferrel", 
        level: "Grill / Mediterran", 
        detail: "Grill / mediterran / portugiesisch — gute Fleisch- und Fischgerichte, aber auch oft gerecht für Gruppen und flexible Gäste.",
        wifi: "Verfügbar",
        rating: 4.0 
    },
    { 
        id: 77, 
        name: "Cartel 71 Taqueria & Tequila Bar", 
        type: "Restaurants", 
        zone: "Ferrel", 
        level: "Mexikanisch / Tapas", 
        detail: "Mexikanisch / Lateinamerika / Tapas-Bar mit günstigen Margaritas; für Abende mit Freunden oder Gruppen-Food gut geeignet.",
        wifi: "Gut",
        rating: 4.3 
    },
    { 
        id: 78, 
        name: "Sushi Fish Baleal", 
        type: "Restaurants", 
        zone: "Baleal", 
        level: "Japanisch (Sushi)", 
        detail: "Sushi/Japanisch — gute Bewertungen, alternative zum Standard Surf-Food; geeignet wenn man Lust auf was anderes hat.",
        wifi: "Stabil",
        rating: 4.5 
    },
    { 
        id: 79, 
        name: "HOWM Baleal", 
        type: "Restaurants", 
        zone: "Baleal", 
        level: "Café/Brunch (Veggie-Snacks)", 
        detail: "Café/Bar mit entspannter Atmosphäre, ideal für Frühstück/Brunch oder „Arbeiten & Relaxen“ mit WLAN.",
        wifi: "Sehr verlässlich",
        rating: 4.9 
    },
    { 
        id: 80, 
        name: "Kalifood", 
        type: "Restaurants", 
        zone: "Ferrel", 
        level: "Frühstück/Snacks (Veggie-Optionen)", 
        detail: "Café/Restaurant mit Frühstück, Snacks und oft vegetarischen Optionen — beliebt für entspannte Mahlzeiten oder Brunch.",
        wifi: "Stabil",
        rating: 4.7 
    },
    { 
        id: 81, 
        name: "Celeiro Café", 
        type: "Restaurants", 
        zone: "Peniche", 
        level: "Gesunde, aufmerksame Küche (Veggie)", 
        detail: "Gesund-es, lockeres Café mit aufmerksamer Küche — gut für Mittagessen oder Kaffee + Snack, auch für Vegetarier geeignet.",
        wifi: "Gut",
        rating: 4.5 
    },
    { 
        id: 82, 
        name: "Roots Café Peniche", 
        type: "Restaurants", 
        zone: "Peniche", 
        level: "🌱 Vegan/Vegetarisch freundlich", 
        detail: "Vegan/vegetarisch freundlich, gute Lattes & frische Säfte – eine Alternative für Plant-based Reisende.",
        wifi: "Ruhig",
        rating: 4.8 
    },
    { 
        id: 83, 
        name: "Taberna do Ganhão", 
        type: "Restaurants", 
        zone: "Ferrel", 
        level: "Meeresfrüchte & Mediterran", 
        detail: "Meeresfrüchte & mediterrane Küche – gute Reviews, nette Atmosphäre, häufig von Surfern besucht.",
        wifi: "Verfügbar",
        rating: 4.4 
    },
    { 
        id: 84, 
        name: "Pateo Da Vila", 
        type: "Restaurants", 
        zone: "Ferrel", 
        level: "Europäisch/Portugiesisch", 
        detail: "Europäisch/portugiesisch, etwas außerhalb der Hauptstraße – oft gutes Essen und ruhige, entspannte Atmosphäre.",
        wifi: "Verfügbar",
        rating: 4.3 
    },
    { id: 85, name: "MastersBurger Ferrel", type: "Restaurants", zone: "Ferrel", level: "Burger & Fast-Food", detail: "Burger & Fast-Food, gutes Preis-Leistungs-Verhältnis — spaßige, unkomplizierte Option nach Surf oder Nachtleben.", wifi: "Verfügbar", rating: 3.9 },
    { id: 86, name: "Funky Donkey Pizza", type: "Restaurants", zone: "Ferrel", level: "Holzofen-Pizza", detail: "Pizza aus dem Holzofen — praktisch für Gruppen oder spät abends, wenn man’s unkompliziert mag.", wifi: "Gut", rating: 4.1 },
    { id: 87, name: "Officina do Hamburger", type: "Restaurants", zone: "Peniche", level: "Hamburger / International", detail: "Hamburger / internationale Küche — einfache, schnelle Lösung für einen Hunger zwischendurch.", wifi: "Verfügbar", rating: 3.6 },
    { id: 88, name: "Restaurante Bateira", type: "Restaurants", zone: "Peniche", level: "Portugiesisch / Europäisch", detail: "Portugiesisch / europäisch, gutes Preis-Leistungs-Verhältnis und eine Option, wenn man etwas mehr Auswahl möchte als Fast Food.", wifi: "Verfügbar", rating: 4.0 },
    { id: 89, name: "Il Boccone", type: "Restaurants", zone: "Peniche", level: "Italienisch / Pizza", detail: "Italienisch/Pizza – günstige Preise, gute Bewertungen, gute Wahl zum Mitnehmen oder entspannten Essen.", wifi: "Verfügbar", rating: 4.2 },
    { id: 90, name: "Regatta", type: "Restaurants", zone: "Peniche", level: "Bar & Pizza", detail: "Bar & Pizza – einfache, entspannte Küche; praktisch wenn man etwas unkompliziertes & preiswertes sucht.", wifi: "Verfügbar", rating: 3.7 },
    { id: 91, name: "3House Beach Bar", type: "Restaurants", zone: "Baleal", level: "Internationales Frühstück/Lunch", detail: "Strandbar mit internationalem Frühstück/Lunch, oft mit vegetarischen Optionen — gute entspannte Wahl für Mittag am Meer.", wifi: "Gut", rating: 4.6 },
    { id: 92, name: "Surfers Lodge Restaurant", type: "Restaurants", zone: "Baleal", level: "Stilvoll, Surfer-Lifestyle", detail: "Stilvolles Restaurant im Surf-Hotel-Setting, gutes Essen, oft mit Besuchern aus aller Welt – gute Mischung aus Komfort & Surfer-Lifestyle.", wifi: "Verfügbar", rating: 4.8 },
];

// Statische Event-Daten (STABIL: Wird jetzt wieder direkt für die Kalenderansicht verwendet)
const EVENTS = [
    { name: "Open Mic Night", location: "Washed Up", day: "Montag", time: "21:00 Uhr", category: "Kultur/Music", rsvps: [] },
    { name: "Portugiesisch Stammtisch", location: "Café Pingo Doce", day: "Montag", time: "18:00 Uhr", category: "Kultur/Essen", rsvps: ['user1', 'user2'] },
    { name: "Beachvolleyball Turnier", location: "Baleal Beach", day: "Montag", time: "16:00 Uhr", category: "Sport/Meetup", rsvps: ['user1', 'user3', 'user4'] },
    { name: "Surf Meetup Anfänger", location: "Ericeira Hub", day: "Dienstag", time: "10:00 Uhr", category: "Sport/Meetup", rsvps: ['user5'] },
    { name: "Marketing Workshop: SEO Basics", location: "Cowork Baleal", day: "Dienstag", time: "14:00 Uhr", category: "Networking", rsvps: [] },
    { name: "Tech Founders Lunch", location: "Cowork Baleal", day: "Mittwoch", time: "13:00 Uhr", category: "Networking", rsvps: ['user1'] },
    { name: "Live Music", location: "Ricle", day: "Mittwoch", time: "22:00 Uhr", category: "Kultur/Music", rsvps: ['user2', 'user3'] },
    { name: "Sunset Drinks & Digital Hangout", location: "Sunset Bar Ericeira", day: "Donnerstag", time: "18:30 Uhr", category: "Soziales", rsvps: ['user4'] },
    { name: "Sundowner Barbecue", location: "Peniche Beach", day: "Freitag", time: "19:00 Uhr", category: "Soziales", rsvps: [] },
    { name: "Global Kitchen Kochevent & Dinner", location: "Gemeinschaftsküche Peniche", day: "Freitag", time: "20:00 Uhr", category: "Kultur/Essen", rsvps: ['user5', 'user1'] },
    { name: "Lissabon Tagesausflug", location: "Peniche (Treffpunkt)", day: "Samstag", time: "08:30 Uhr", category: "Kultur/Ausflug", rsvps: ['user3'] },
    { name: "Yoga am Meer", location: "Foz do Lizandro", day: "Sonntag", time: "09:00 Uhr", category: "Sport/Meetup", rsvps: ['user2'] },
    { name: "Strandreinigung & Conservation Talk", location: "Supertubos Beach", day: "Sonntag", time: "14:00 Uhr", category: "Soziales/Umwelt", rsvps: [] },
];

// Hilfsfunktion zur Kategorisierung und Icon-Auswahl
const getCategoryIcon = (category) => {
    if (!category) return '✨';
    if (category.includes('Surf') || category.includes('Sport') || category.includes('Yoga')) return '🏄';
    if (category.includes('Networking') || category.includes('Tech') || category.includes('Founders') || category.includes('Marketing')) return '🤝';
    if (category.includes('Kultur') || category.includes('Ausflug') || category.includes('Stammtisch') || category.includes('Music')) return '🏛️';
    if (category.includes('Essen') || category.includes('Barbecue') || category.includes('Lunch') || category.includes('Dinner')) return '🍽️';
    if (category.includes('Umwelt') || category.includes('Conservation')) return '♻️'; 
    return '✨';
};

// Exponentielles Backoff für API-Anfragen
const fetchWithRetry = async (url, options, retries = 3) => {
    for (let i = 0; i < retries; i++) {
        try {
            const response = await fetch(url, options);
            if (!response.ok) throw new Error(`HTTP error! status: ${response.status}`);
            return response;
        } catch (error) {
            if (i < retries - 1) {
                const delay = Math.pow(2, i) * 1000;
                await new Promise(resolve => setTimeout(resolve, delay));
            } else {
                throw error;
            }
        }
    }
};

const App = () => {
    const [activeTab, setActiveTab] = useState('map');
    const [selectedCategory, setSelectedCategory] = useState('Alle');
    const [messages, setMessages] = useState([]);
    const [newMessage, setNewMessage] = useState('');
    const [db, setDb] = useState(null);
    const [auth, setAuth] = useState(null);
    const [userId, setUserId] = useState(null);
    const [isAuthReady, setIsAuthReady] = useState(false);
    const [chatError, setChatError] = useState(null);
    
    // NEU: State für die Detailansicht
    const [selectedSpot, setSelectedSpot] = useState(null);

    // States für Wetterdaten-Integration
    // LOGIK FÜR WETTERDATEN ENTFERNT
    const [weatherData, setWeatherData] = useState(null);
    const [weatherLoading, setWeatherLoading] = useState(false);
    const [weatherError, setWeatherError] = useState(null);
    
    // State für den FlowBot
    const [agentInput, setAgentInput] = useState('');
    const [agentChat, setAgentChat] = useState([
        { role: 'assistant', text: 'Olá! Ich bin FlowBot, Ihr KI-Assistent für digitale Nomaden in Portugal. Fragen Sie mich nach WLAN, Surf-Cams oder Visabestimmungen!', sources: [] }
    ]);
    const [agentIsLoading, setAgentIsLoading] = useState(false);
    
    // State für Community View Toggle
    const [communityView, setCommunityView] = useState('chat'); // 'chat' oder 'forum'

    const chatEndRef = useRef(null);
    
    // NEU: Events State (zurück auf statisch für Prototyp-Stabilität)
    const [localEvents, setLocalEvents] = useState(EVENTS);
    const [eventSubmissionStatus, setEventSubmissionStatus] = useState(null);

    // -----------------------------------------------------
    // 1. FIREBASE INITIALISIERUNG UND AUTHENTIFIZIERUNG
    // -----------------------------------------------------

    useEffect(() => {
        // Firebase ist nur noch für Chat/Auth/FlowBot nötig, aber die Initialisierung bleibt
        if (Object.keys(firebaseConfig).length === 0 || !firebaseConfig.projectId) {
            console.error("Firebase Config nicht gefunden oder unvollständig. Chat/Forum ist deaktiviert.");
            setChatError("Firebase-Initialisierung fehlgeschlagen.");
            setIsAuthReady(true);
            return;
        }

        try {
            const app = initializeApp(firebaseConfig);
            const authInstance = getAuth(app);
            const dbInstance = getFirestore(app);
            setAuth(authInstance);
            setDb(dbInstance);

            const unsubscribe = onAuthStateChanged(authInstance, async (user) => {
                if (!user) {
                    try {
                        if (initialAuthToken) {
                            // initialAuthToken ist hier der Wert der Konstante aus dem globalen Scope
                            await signInWithCustomToken(authInstance, initialAuthToken);
                        } else {
                            await signInAnonymously(authInstance);
                        }
                    } catch (error) {
                        console.error("Anmeldung fehlgeschlagen:", error);
                        setChatError("Anmeldung fehlgeschlagen. Chat-Funktion nicht verfügbar.");
                    }
                }
                setUserId(authInstance.currentUser?.uid || crypto.randomUUID());
                setIsAuthReady(true);
            });

            return () => unsubscribe();
        } catch (e) {
            console.error("Fehler bei der Firebase-Initialisierung:", e);
            setChatError("Fehler bei der Initialisierung. Chat-Funktion nicht verfügbar.");
            setIsAuthReady(true);
        }
    }, []);

    // -----------------------------------------------------
    // 2. CHAT MESSAGES LADEN (Echtzeit mit onSnapshot)
    // -----------------------------------------------------

    useEffect(() => {
        if (!isAuthReady || !db) return;

        const chatCollectionPath = `/artifacts/${appId}/public/data/portugal_chat`;
        const q = query(
            collection(db, chatCollectionPath),
            limit(50)
        );

        const unsubscribe = onSnapshot(q, (snapshot) => {
            const fetchedMessages = snapshot.docs.map(doc => ({
                id: doc.id,
                ...doc.data()
            }));
            
            // In-Memory Sortierung nach Timestamp (aufsteigend)
            fetchedMessages.sort((a, b) => {
                const timeA = a.timestamp?.seconds || 0;
                const timeB = b.timestamp?.seconds || 0;
                return timeA - timeB; // Aufsteigend sortieren
            });
            
            setMessages(fetchedMessages);
        }, (error) => {
            console.error("Fehler beim Laden der Nachrichten:", error);
            setChatError(`Fehler beim Laden der Nachrichten: ${error.message}. Überprüfen Sie Ihre Firestore Security Rules.`);
        });

        return () => unsubscribe();
    }, [isAuthReady, db, appId]);

    // -----------------------------------------------------
    // NEU: 2B. ECHZEIT WETTER DATEN (Simulierte API-Integration)
    // -----------------------------------------------------
    // LOGIK FÜR WETTERDATEN ENTFERNT

    // -----------------------------------------------------
    // 3. CHAT MESSAGE SENDEN
    // -----------------------------------------------------

    const handleSendMessage = useCallback(async (e) => {
        e.preventDefault();
        if (newMessage.trim() === '' || !db || !userId) return;

        const messageData = {
            text: newMessage.trim(),
            timestamp: serverTimestamp(),
            userId: userId,
            username: `Nomade-${userId.substring(0, 4)}`,
        };

        try {
            const chatCollectionPath = `/artifacts/${appId}/public/data/portugal_chat`;
            await addDoc(collection(db, chatCollectionPath), messageData);
            setNewMessage('');
        } catch (error) {
            console.error("Fehler beim Senden der Nachricht:", error);
            setChatError(`Nachricht konnte nicht gesendet werden: ${error.message}.`);
        }
    }, [newMessage, db, userId, appId]);
    
    // -----------------------------------------------------
    // 4. KI AGENT LOGIC (FlowBot mit Citation Extraction)
    // -----------------------------------------------------
    
    const handleAgentSubmit = useCallback(async (e) => {
        e.preventDefault();
        if (agentInput.trim() === '' || agentIsLoading) return;

        const userQuery = agentInput.trim();
        setAgentChat(prev => [...prev, { role: 'user', text: userQuery, sources: [] }]);
        setAgentInput('');
        setAgentIsLoading(true);

        const apiKey = "";
        const apiUrl = `https://generativelanguage.googleapis.com/v1beta/models/gemini-2.5-flash-preview-09-2025:generateContent?key=${apiKey}`;

        const payload = {
            contents: [{ parts: [{ text: userQuery }] }],
            tools: [{ "google_search": {} }],
            systemInstruction: {
                parts: [{ text: "Du bist FlowBot, ein freundlicher und hilfsbereiter KI-Assistent für digitale Nomaden in Portugal (Peniche, Baleal, Ericeira). Deine Antworten sind kurz, präzise und verwenden, wenn möglich, aktuelle Informationen aus dem Web (Google Search). Sei enthusiastisch und verwende Emojis." }]
            },
        };

        try {
            const response = await fetchWithRetry(apiUrl, {
                method: 'POST',
                headers: { 'Content-Type': 'application/json' },
                body: JSON.stringify(payload)
            });

            const result = await response.json();
            const candidate = result.candidates?.[0];
            const text = candidate?.content?.parts?.[0]?.text || "Entschuldigung, FlowBot konnte keine Antwort finden. Probiere es später noch einmal!😥";
            
            // NEU: Extrahiere Quellen (Citations)
            let sources = [];
            const groundingMetadata = candidate?.groundingMetadata;
            if (groundingMetadata && groundingMetadata.groundingAttributions) {
                sources = groundingMetadata.groundingAttributions
                    .map(attribution => ({
                        uri: attribution.web?.uri,
                        title: attribution.web?.title,
                    }))
                    .filter(source => source.uri && source.title); // Nur gültige Quellen
            }


            setAgentChat(prev => [...prev, { role: 'assistant', text: text, sources: sources }]);
        } catch (error) {
            setAgentChat(prev => [...prev, { role: 'assistant', text: 'Entschuldigung, bei der Kommunikation mit dem Server ist ein Fehler aufgetreten.😔', sources: [] }]);
        } finally {
            setAgentIsLoading(false);
        }
    }, [agentInput, agentIsLoading]);


    // -----------------------------------------------------
    // 5. EVENT DATEN UND SUBMISSION (Simuliert)
    // -----------------------------------------------------
    
    // Fügt ein Event lokal zum State hinzu (SIMULATION)
    const handleAddEvent = (newEvent) => {
        // Generiere eine temporäre ID für das lokale Event
        const newEventWithId = {
            ...newEvent,
            id: Date.now() + Math.random(),
            time: newEvent.time || 'Unbestimmt',
            category: newEvent.category || 'Soziales',
            location: newEvent.location || 'Unbestimmt',
            rsvps: [userId], // Der Ersteller nimmt teil
        };

        setLocalEvents(prevEvents => {
            const combinedEvents = [...prevEvents, newEventWithId];
            
            const dayOrder = ["Montag", "Dienstag", "Mittwoch", "Donnerstag", "Freitag", "Samstag", "Sonntag"];
            
            combinedEvents.sort((a, b) => {
                const dayA = dayOrder.indexOf(a.day);
                const dayB = dayOrder.indexOf(b.day);
                
                if (dayA !== dayB) return dayA - dayB;
                return String(a.time || '00:00').localeCompare(String(b.time || '00:00'));
            });

            return combinedEvents;
        });

        // Simuliere Erfolg
        setEventSubmissionStatus({ success: true, message: 'Event erfolgreich eingereicht! Es erscheint lokal in deinem Kalender (Beachte: Bei Neuladen der App geht dieses Event verloren, da die Datenbank nicht verwendet wird).' });
        setTimeout(() => setEventSubmissionStatus(null), 8000);
    };
    
    // NEU: RSVP Handler
    const handleRsvp = (eventId) => {
        if (!userId) return; // Sollte nicht passieren, da authReady geprüft wird

        setLocalEvents(prevEvents => prevEvents.map(event => {
            if (event.id === eventId) {
                const isRsvped = event.rsvps.includes(userId);
                
                const newRsvps = isRsvped
                    ? event.rsvps.filter(id => id !== userId) // Abmelden
                    : [...event.rsvps, userId]; // Anmelden

                return { ...event, rsvps: newRsvps };
            }
            return event;
        }));
    };


    // Hilfsfunktion zum Gruppieren der Events nach Tag (Nutzt den lokalen State)
    const eventsByDayLocal = localEvents.reduce((acc, event) => {
        const day = event.day || 'Unbekannt';
        if (!acc[day]) {
            acc[day] = [];
        }
        acc[day].push(event);
        return acc;
    }, {});


    // -----------------------------------------------------
    // 6. FILTER LOGIK FÜR DIE MAP
    // -----------------------------------------------------

    // Spot Map Filter
    const filteredLocations = LOCATIONS.filter(location => {
        
        // Filtert nach Surf-Spot, Coworking oder Cafés
        const categoryMatch = selectedCategory === 'Alle' || location.type === selectedCategory;
        
        // Filtert nach Favoriten (Rating >= 4.5)
        const isFavorite = selectedCategory === 'Favoriten' && location.rating >= 4.5;

        return categoryMatch || isFavorite;
    });

    // -----------------------------------------------------
    // 7. RENDER KOMPONENTEN
    // -----------------------------------------------------

    // NEU: Event Einreichungsformular Komponente
    const EventSubmitForm = ({ onSubmit, status }) => {
        const [name, setName] = useState('');
        const [location, setLocation] = useState('');
        const [day, setDay] = useState('Montag');
        const [time, setTime] = useState('18:00 Uhr');
        const [category, setCategory] = useState('Soziales');

        const handleSubmit = (e) => {
            e.preventDefault();
            onSubmit({ name, location, day, time, category });
            setName('');
            setLocation('');
            setTime('18:00 Uhr');
        };

        const days = ["Montag", "Dienstag", "Mittwoch", "Donnerstag", "Freitag", "Samstag", "Sonntag"];
        const categories = ["Soziales", "Networking", "Sport/Meetup", "Kultur/Music", "Kultur/Essen", "Kultur/Ausflug", "Umwelt"];

        return (
            <div className="bg-white p-4 rounded-xl shadow border border-cyan-100 mb-4">
                <h3 className="text-xl font-bold text-cyan-700 mb-3">Community Event hinzufügen</h3>
                {status && (
                    <div className={`p-2 text-sm rounded-lg mb-3 ${status.success ? 'bg-green-100 text-green-700' : 'bg-red-100 text-red-700'}`}>
                        {status.message}
                    </div>
                )}
                
                <form onSubmit={handleSubmit} className="space-y-3">
                    <input 
                        type="text" 
                        placeholder="Event Name (z.B. 'Sunset Drinks')" 
                        value={name} 
                        onChange={(e) => setName(e.target.value)}
                        className="w-full p-2 border border-stone-300 rounded-lg focus:ring-cyan-500"
                        required
                    />
                    <input 
                        type="text" 
                        placeholder="Location (z.b. 'Washed Up Café')" 
                        value={location} 
                        onChange={(e) => setLocation(e.target.value)}
                        className="w-full p-2 border border-stone-300 rounded-lg focus:ring-cyan-500"
                        required
                    />
                    
                    <div className="grid grid-cols-3 gap-3">
                        <select value={day} onChange={(e) => setDay(e.target.value)} 
                                className="col-span-1 p-2 border border-stone-300 rounded-lg focus:ring-cyan-500">
                            {days.map(d => <option key={d} value={d}>{d}</option>)}
                        </select>
                        <input 
                            type="text" 
                            placeholder="Uhrzeit (HH:MM)" 
                            value={time} 
                            onChange={(e) => setTime(e.target.value)}
                            className="col-span-1 p-2 border border-stone-300 rounded-lg focus:ring-cyan-500"
                            required
                        />
                        <select value={category} onChange={(e) => setCategory(e.target.value)} 
                                className="col-span-1 p-2 border border-stone-300 rounded-lg focus:ring-cyan-500 text-xs">
                            {categories.map(c => <option key={c} value={c}>{c.split('/')[0]}</option>)}
                        </select>
                    </div>

                    <button 
                        type="submit" 
                        className="w-full p-3 bg-cyan-600 text-white font-semibold rounded-xl hover:bg-cyan-700 transition-colors shadow-md"
                    >
                        Event posten
                    </button>
                </form>
            </div>
        );
    };

    // NEU: Detailansicht für Spots (Sub-Page Simulation)
    const renderSpotDetail = () => (
        <div className="p-4 space-y-4">
            <button 
                onClick={() => setSelectedSpot(null)}
                className="flex items-center text-cyan-600 hover:text-cyan-800 transition-colors mb-4 p-2 -ml-2 rounded-lg"
            >
                <svg className="w-5 h-5 mr-1" fill="none" stroke="currentColor" viewBox="0 0 24 24" xmlns="http://www.w3.org/2000/svg"><path strokeLinecap="round" strokeLinejoin="round" strokeWidth="2" d="M10 19l-7-7m0 0l7-7m-7 7h18"></path></svg>
                Zurück zur Spot-Liste
            </button>
            
            <div className="bg-white p-6 rounded-2xl shadow-xl border border-stone-200 space-y-4">
                <h1 className="text-3xl font-extrabold text-cyan-800">{selectedSpot.name}</h1>
                
                <div className="flex items-center space-x-4 text-lg">
                    <p className="font-semibold text-stone-700">Typ:</p>
                    <span className="text-cyan-600 font-bold">{selectedSpot.type}</span>
                </div>
                
                {/* Zeigt Level oder WLAN-Geschw. */}
                <div className="flex items-center space-x-4 text-lg">
                    <p className="font-semibold text-stone-700">{selectedSpot.type === 'Surf-Spot' ? 'Level:' : 'WLAN:'}</p>
                    <span className={`font-bold ${selectedSpot.type === 'Surf-Spot' ? 'text-amber-600' : 'text-green-600'}`}>
                        {selectedSpot.level || selectedSpot.wifi || 'N/A'}
                    </span>
                </div>
                
                <p className="text-sm text-stone-600 leading-relaxed">{selectedSpot.detail}</p>

                {selectedSpot.type === 'Surf-Spot' && (
                    <>
                        <h3 className="text-xl font-bold pt-2 border-t mt-4 text-stone-700">🌊 Optimale Bedingungen</h3>
                        
                        <div className="grid grid-cols-2 gap-4 text-sm">
                            <div className="p-3 bg-stone-50 rounded-lg border border-stone-200">
                                <p className="font-semibold text-cyan-700">Best Wind</p>
                                <p className="text-stone-600">{selectedSpot.bestWind || 'N/A'}</p>
                            </div>
                            <div className="p-3 bg-stone-50 rounded-lg border border-stone-200">
                                <p className="font-semibold text-cyan-700">Best Swell</p>
                                <p className="text-stone-600">{selectedSpot.bestSwell || 'N/A'}</p>
                            </div>
                        </div>
                    </>
                )}
                
                {selectedSpot.type === 'Coworking' || selectedSpot.type === 'Cafés' || selectedSpot.type === 'Bars' || selectedSpot.type === 'Restaurants' ? ( // Typ 'Restaurants' prüfen
                    <>
                        <h3 className="text-xl font-bold pt-2 border-t mt-4 text-stone-700">💻 Details</h3>
                        
                        <div className="grid grid-cols-2 gap-4 text-sm">
                            <div className="p-3 bg-stone-50 rounded-lg border border-stone-200">
                                <p className="font-semibold text-cyan-700">WLAN Geschwindigkeit</p>
                                <p className="text-stone-600">{selectedSpot.wifi || 'N/A'}</p>
                            </div>
                            <div className="p-3 bg-stone-50 rounded-lg border border-stone-200">
                                <p className="font-semibold text-cyan-700">Atmosphäre / Kurzbeschreibung</p>
                                <p className="text-stone-600">{selectedSpot.level || 'Nicht verfügbar'}</p>
                            </div>
                        </div>
                    </>
                ) : null}


                <div className="mt-4 flex items-center justify-between text-sm pt-4 border-t">
                    <p className="text-stone-500">Zone: {selectedSpot.zone}</p>
                    <span className="flex items-center text-amber-500 font-bold">
                        {selectedSpot.rating} <svg className="w-4 h-4 ml-1" fill="currentColor" viewBox="0 0 20 20" xmlns="http://www.w3.org/2000/svg"><path d="M9.049 2.927c.3-.921 1.603-.921 1.902 0l1.071 3.292 3.486.254c.96.07 1.348 1.258.646 1.909l-2.671 2.37.818 3.328c.277 1.133-.941 2.029-1.839 1.419L10 15.658l-2.923 1.952c-.898.61-2.116-.286-1.839-1.419l.818-3.328-2.671-2.37c-.702-.651-.314-1.839.646-1.909l3.486-.254 1.071-3.328z"></path></svg>
                    </span>
                </div>
            </div>
        </div>
    );


    // Forum Tab Content
    const renderForumContent = () => (
        <div className="flex flex-col h-full bg-stone-50">
            <h2 className="text-2xl font-extrabold p-4 pb-2 text-cyan-700 border-b border-stone-200">
                <span className="text-xl mr-2">💬</span>Community Hub
            </h2>

            {/* Toggle Switch für Chat/Forum */}
            <div className="p-4 pt-2 border-b border-stone-200 bg-white">
                <div className="flex bg-stone-200 rounded-full p-1 shadow-inner">
                    <button
                        onClick={() => setCommunityView('chat')}
                        className={`flex-1 p-2 text-sm font-semibold rounded-full transition-all ${
                            communityView === 'chat' ? 'bg-cyan-600 text-white shadow-md' : 'text-stone-600 hover:bg-stone-300'
                        }`}
                    >
                        Live Chat
                    </button>
                    <button
                        onClick={() => setCommunityView('forum')}
                        className={`flex-1 p-2 text-sm font-semibold rounded-full transition-all ${
                            communityView === 'forum' ? 'bg-cyan-600 text-white shadow-md' : 'text-stone-600 hover:bg-stone-300'
                        }`}
                    >
                        Expats Forum
                    </button>
                </div>
            </div>

            <div className="p-4 space-y-4">
                <h3 className="text-xl font-bold text-stone-700">Wichtige Foren-Threads (Simulation)</h3>
                <p className="text-sm text-stone-500">Strukturierter Austausch zu komplexen Themen für Langzeit-Nomaden und Expats.</p>

                {[
                    { title: "Visum D7 & Steuern für Freiberufler", replies: 42, last: "André C. (Heute 15:30)" },
                    { title: "Langzeit-Mietwohnungen in Ericeira (6+ Monate)", replies: 19, last: "Leonie S. (Gestern 10:15)" },
                    { title: "Empfehlungen für portugiesische Bankkonten (NIF)", replies: 88, last: "Admin (Gestern 09:00)" },
                    { title: "Beste Coworking Spaces mit privatem Büro", replies: 5, last: "Tech Nomade (vor 2h)" },
                ].map((thread, index) => (
                    <div key={index} className="bg-white p-4 rounded-xl shadow border border-stone-200 transition-shadow duration-300 hover:shadow-cyan-200 cursor-pointer">
                        <h4 className="text-lg font-semibold text-cyan-700">{thread.title}</h4>
                        <div className="flex justify-between text-xs mt-1 text-stone-500">
                            <span>Antworten: {thread.replies}</span>
                            <span>Letzte Aktivität: {thread.last}</span>
                        </div>
                    </div>
                ))}
                <button className="w-full mt-4 p-3 bg-cyan-100 text-cyan-700 font-semibold rounded-xl hover:bg-cyan-200 transition-colors">
                    Neuen Thread erstellen (Simulation)
                </button>
            </div>
        </div>
    );

    // Community Tab (Live Chat + Forum Toggle)
    const renderCommunityTab = () => (
        <div className="flex flex-col h-full bg-stone-50">
            <h2 className="text-2xl font-extrabold p-4 pb-2 text-cyan-700 border-b border-stone-200">
                <span className="text-xl mr-2">💬</span>Community Hub
            </h2>

            {/* Toggle Switch für Chat/Forum */}
            <div className="p-4 pt-2 border-b border-stone-200 bg-white">
                <div className="flex bg-stone-200 rounded-full p-1 shadow-inner">
                    <button
                        onClick={() => setCommunityView('chat')}
                        className={`flex-1 p-2 text-sm font-semibold rounded-full transition-all ${
                            communityView === 'chat' ? 'bg-cyan-600 text-white shadow-md' : 'text-stone-600 hover:bg-stone-300'
                        }`}
                    >
                        Live Chat
                    </button>
                    <button
                        onClick={() => setCommunityView('forum')}
                        className={`flex-1 p-2 text-sm font-semibold rounded-full transition-all ${
                            communityView === 'forum' ? 'bg-cyan-600 text-white shadow-md' : 'text-stone-600 hover:bg-stone-300'
                        }`}
                    >
                        Expats Forum
                    </button>
                </div>
            </div>

            {/* Content Rendering */}
            {communityView === 'chat' ? renderLiveChatContent() : renderForumContent()}

        </div>
    );

    // Live Chat Content (Extrahiert aus dem alten renderChatTab)
    const renderLiveChatContent = () => (
        <>
            {chatError && <div className="p-3 text-sm text-red-700 bg-red-100">{chatError}</div>}

            <div className="p-2 text-xs text-stone-500 bg-stone-100 border-b border-stone-200">
                Ihre Nutzer-ID (Für Multi-User): <span className="font-mono text-stone-700">{userId || 'Wird geladen...'}</span>
            </div>
            
            <div className="flex-1 overflow-y-auto p-4 space-y-3" style={{ maxHeight: 'calc(100vh - 200px - 90px)' }}>
                {messages.length === 0 && isAuthReady && <p className="text-center text-stone-400">Seien Sie der Erste im Chat! 👋</p>}
                {messages.map(msg => (
                    <div 
                        key={msg.id} 
                        className={`flex ${msg.userId === userId ? 'justify-end' : 'justify-start'}`}
                    >
                        <div className={`max-w-xs sm:max-w-sm p-3 rounded-xl shadow-md ${
                            msg.userId === userId 
                                ? 'bg-cyan-600 text-white rounded-br-none' 
                                : 'bg-white text-stone-800 rounded-tl-none border border-stone-200' 
                        }`}>
                            <span className={`block text-xs font-semibold mb-1 opacity-80 ${msg.userId === userId ? 'text-cyan-100' : 'text-stone-500'}`}>
                                {msg.userId === userId ? 'Sie' : msg.username || 'Unbekannter Nomade'}
                            </span>
                            <p className='break-words'>{msg.text}</p>
                            <span className="block text-xs mt-1 opacity-60">
                                {msg.timestamp ? new Date(msg.timestamp.seconds * 1000).toLocaleTimeString() : '...'}
                            </span>
                        </div>
                    </div>
                ))}
                <div ref={chatEndRef} />
            </div>

            <form onSubmit={handleSendMessage} className="p-4 border-t border-stone-200 bg-white">
                <div className="flex space-x-2">
                    <input
                        type="text"
                        value={newMessage}
                        onChange={(e) => setNewMessage(e.target.value)}
                        placeholder="Schreiben Sie eine Nachricht..."
                        className="flex-1 p-3 border border-stone-300 rounded-lg focus:ring-cyan-500 focus:border-cyan-500"
                        disabled={!isAuthReady || !db}
                    />
                    <button
                        type="submit"
                        className="px-4 py-3 bg-cyan-600 text-white rounded-xl font-semibold hover:bg-cyan-700 disabled:bg-cyan-300 transition-colors shadow-md shadow-cyan-300/50"
                        disabled={!isAuthReady || !db || newMessage.trim() === ''}
                    >
                        Senden
                    </button>
                </div>
            </form>
        </>
    );
    
    // Alte renderChatTab wird durch renderCommunityTab ersetzt
    const renderChatTab = renderCommunityTab;
    
    const renderMapTab = () => {
        // Wenn ein Spot ausgewählt ist, zeige die Detailseite
        if (selectedSpot) {
            return renderSpotDetail();
        }

        // Andernfalls zeige die Liste
        return (
            <div className="p-4 space-y-4">
                <h2 className="text-2xl font-extrabold text-cyan-700 tracking-tight">Peniche & Baleal</h2>
                <p className="text-sm text-stone-600">Finde Coworking Spaces, Cafés und Surf-Spots in deiner Nähe.</p>
                
                {/* Dynamische Wetterdaten-Anzeige */}
                <div className="w-full h-48 bg-stone-200 rounded-xl overflow-hidden shadow-inner flex items-center justify-center relative border border-stone-300">
                    <img 
                        src="https://placehold.co/800x600/67E8F9/0F172A?text=Peniche+%26+Baleal+Surf+Map" 
                        alt="Visuelle Kartendarstellung der Peniche/Baleal Region" 
                        className="w-full h-full object-cover opacity-80"
                    />
                    <div className="absolute top-0 left-0 w-full h-full flex items-center justify-center bg-cyan-900 bg-opacity-10">
                        <p className="text-white text-lg font-bold drop-shadow-lg">Kartenansicht (Simulation)</p>
                    </div>
                </div>

                {/* Kategorien-Filter */}
                <h3 className="text-lg font-semibold text-stone-700 mt-4 border-t pt-4">Spot-Kategorien</h3>
                <div className="flex space-x-2 overflow-x-auto pb-2 -mx-4 px-4">
                    {['Alle', 'Surf-Spot', 'Coworking', 'Cafés', 'Restaurants', 'Bars'].map(cat => ( // Alle 5 Kategorien + Alle
                        <button
                            key={cat}
                            onClick={() => setSelectedCategory(cat)}
                            className={`px-4 py-2 text-sm font-medium rounded-full transition-all duration-200 whitespace-nowrap shadow-md ${
                                selectedCategory === cat ? 'bg-cyan-600 text-white shadow-cyan-300/50' : 'bg-white text-stone-700 hover:bg-cyan-50 border border-stone-200'
                            }`}
                        >
                            {cat}
                        </button>
                    ))}
                </div>

                {/* Ergebnisliste (Umgestellt auf neues Format) */}
                <div className="space-y-3 pt-2">
                    <p className="text-sm text-stone-500">{filteredLocations.length} Spots gefunden ({selectedCategory})</p>
                    {filteredLocations.length > 0 ? (
                        filteredLocations.map(loc => (
                            <div 
                                key={loc.id} 
                                className="bg-white p-4 rounded-xl shadow-lg border border-stone-100 transition-shadow duration-300 hover:shadow-cyan-200 cursor-pointer"
                                onClick={() => setSelectedSpot(loc)} // NEU: Klick öffnet Detailansicht
                            >
                                {/* NEUES FORMAT: Name fett + Level/WLAN Zeile */}
                                <h3 className="text-lg font-bold text-cyan-700">{loc.name}</h3>
                                <p className="text-sm text-stone-600 mt-0.5">
                                    <span className="font-medium text-stone-700">
                                        {loc.type === 'Surf-Spot' ? 'Level:' : loc.type === 'Coworking' || loc.type === 'Restaurants' ? 'WLAN:' : 'Atmosphäre:'}
                                    </span> 
                                    {loc.level || loc.wifi}
                                </p>
                                
                                <p className="text-xs text-stone-500 mt-1 truncate">{loc.detail}</p>
                                
                                <div className="flex items-center space-x-4 text-xs mt-2 text-cyan-600">
                                    <span className="flex items-center">
                                        {/* Icon zur Verdeutlichung der Kategorie */}
                                        <svg className="w-4 h-4 mr-1 text-cyan-500" fill="none" stroke="currentColor" viewBox="0 0 24 24" xmlns="http://www.w3.org/2000/svg"><path strokeLinecap="round" strokeLinejoin="round" strokeWidth="2" d="M13 10V3L4 14h7v7l9-11h-7z"></path></svg>
                                        {loc.zone}
                                    </span>
                                    {/* Die Rating-Anzeige bleibt erhalten */}
                                    <span className="flex items-center text-amber-500">
                                        <svg className="w-4 h-4 mr-1" fill="currentColor" viewBox="0 0 20 20" xmlns="http://www.w3.org/2000/svg"><path d="M9.049 2.927c.3-.921 1.603-.921 1.902 0l1.071 3.292 3.486.254c.96.07 1.348 1.258.646 1.909l-2.671 2.37.818 3.328c.277 1.133-.941 2.029-1.839 1.419L10 15.658l-2.923 1.952c-.898.61-2.116-.286-1.839-1.419l.818-3.328-2.671-2.37c-.702-.651-.314-1.839.646-1.909l3.486-.254 1.071-3.292z"></path></svg>
                                        {loc.rating}
                                    </span>
                                </div>
                                <p className="text-[10px] text-stone-400 mt-1">Klicken für {loc.type === 'Surf-Spot' ? 'Wind-/Swell-Details' : 'Details'}</p>
                            </div>
                        ))
                    ) : (
                        <p className="text-center text-stone-500 mt-8">Keine Spots gefunden, die den Filtern entsprechen.</p>
                    )}
                </div>
            </div>
        );
    };

    const renderEventsTab = () => {
        // Hilfsfunktion zum Gruppieren der Events nach Tag (Nutzt den lokalen State)
        const eventsByDayLocal = localEvents.reduce((acc, event) => {
            const day = event.day || 'Unbekannt';
            if (!acc[day]) {
                acc[day] = [];
            }
            acc[day].push(event);
            return acc;
        }, {});
        
        // Definiere die Reihenfolge der Wochentage für die korrekte Anzeige
        const dayOrder = ["Montag", "Dienstag", "Mittwoch", "Donnerstag", "Freitag", "Samstag", "Sonntag"];
        
        // Clientseitige Sortierung der Tage
        const sortedDays = Object.keys(eventsByDayLocal).sort((a, b) => dayOrder.indexOf(a) - dayOrder.indexOf(b));

        return (
            <div className="p-4 space-y-4">
                <h2 className="text-2xl font-extrabold text-cyan-700 tracking-tight">📅 Events diese Woche <span className="text-lg text-stone-500 ml-2">({getWeeklyDateRange()})</span></h2>
                
                {/* NEU: Event-Einreichungsformular */}
                <EventSubmitForm onSubmit={handleAddEvent} status={eventSubmissionStatus} />
                
                <p className="text-xs text-stone-500 border-b pb-2">Datenquelle: Statische Beispieldaten & lokal eingereichte Events</p>

                <div className="space-y-6">
                    {sortedDays.length > 0 ? (
                        sortedDays.map(day => (
                            <div key={day} className="bg-white p-3 rounded-xl shadow-lg border border-stone-200">
                                <h3 className="text-lg font-bold text-stone-700 border-b border-cyan-100 pb-1 mb-2">
                                    {day}
                                </h3>
                                <div className="space-y-2">
                                    {eventsByDayLocal[day].map((event, index) => {
                                        const userIsRsvped = event.rsvps.includes(userId);

                                        // Kurznamen für die Anzeige der Teilnehmer
                                        const rsvpCount = event.rsvps.length;
                                        const attendeeNames = event.rsvps
                                            .map(id => (id === userId ? 'Du' : `Nomade-${id.substring(0, 4)}`))
                                            .slice(0, 3)
                                            .join(', ');

                                        return (
                                            <div key={index} className="flex items-start space-x-3 p-2 bg-stone-50 rounded-lg transition-colors duration-200 hover:bg-cyan-50">
                                                <div className="text-xl pt-0.5">{getCategoryIcon(event.category)}</div>
                                                <div className="flex-1">
                                                    <p className="text-sm font-semibold text-cyan-800">{event.name}</p>
                                                    <p className="text-xs text-stone-600">
                                                        {event.time} in {event.location}
                                                    </p>
                                                    
                                                    {/* RSVP Anzeige */}
                                                    {rsvpCount > 0 && (
                                                        <p className="text-xs mt-1 text-stone-500 flex items-center">
                                                            <span className="mr-1">👥</span>
                                                            {userIsRsvped && rsvpCount === 1 ? 'Du nimmst teil!' :
                                                             userIsRsvped ? `Du und ${rsvpCount - 1} weitere` :
                                                             `${rsvpCount} Nomaden sind dabei!`}
                                                        </p>
                                                    )}
                                                </div>

                                                <button
                                                    onClick={() => handleRsvp(event.id)}
                                                    className={`text-xs px-3 py-1 rounded-full font-medium transition-colors border ${
                                                        userIsRsvped 
                                                        ? 'bg-green-600 text-white border-green-600 hover:bg-green-700' 
                                                        : 'bg-white text-cyan-600 border-cyan-600 hover:bg-cyan-50'
                                                    }`}
                                                >
                                                    {userIsRsvped ? 'Teilnahme ✓' : 'RSVP'}
                                                </button>
                                            </div>
                                        )
                                    })}
                                </div>
                            </div>
                        ))
                    ) : (
                        <p className="text-center text-stone-500 mt-8">Keine Events für diese Woche in der Liste (Bitte Konstante EVENTS im Code befüllen).</p>
                    )}
                </div>
            </div>
        );
    };

    const renderCamsTab = () => {
        const windInfo = weatherData ? getWindDirection(weatherData.wind.deg) : null;

        return (
            <div className="p-4 space-y-4">
                <h2 className="text-2xl font-extrabold text-cyan-700 tracking-tight">🌊 Surf-Cams & Conditions</h2>
                <p className="text-sm text-stone-600">Live-Bilder der besten Spots: Dein direkter Draht zur perfekten Welle! 🏄‍♂️</p>
                
                {/* HIER PLATZIERT: Dynamische Wetterdaten-Anzeige */}
                <div className="bg-cyan-50 p-4 rounded-xl shadow-inner border border-cyan-200">
                    <h3 className="text-lg font-bold text-cyan-700 mb-2">Aktuelles Wetter (Peniche)</h3>
                    {weatherLoading ? (
                        <p className="text-sm text-cyan-600">Daten werden geladen...</p>
                    ) : weatherError ? (
                        <p className="text-sm text-red-600">Wetterdatenfehler: {weatherError}</p>
                    ) : weatherData ? (
                        <div className="grid grid-cols-2 gap-4 text-stone-700">
                            {/* Temperatur/Beschreibung */}
                            <div className="flex items-center space-x-2">
                                <span className="text-3xl">🌡️</span> 
                                <div>
                                    {/* FIX: Korrekte Syntax für den JSX-Ausdruck */}
                                    <p className="text-xl font-bold">{Math.round(weatherData.main.temp)}°C</p>
                                    <p className="text-xs text-stone-500">{weatherData.weather[0].description}</p>
                                </div>
                            </div>
                            {/* Windrichtung/Stärke */}
                            <div className="text-right">
                                <p className="text-sm font-semibold text-stone-700">Wind: {Math.round(weatherData.wind.speed * 3.6)} km/h</p>
                                <p className={`text-sm font-bold ${windInfo.color}`}>{windInfo.display}</p>
                            </div>
                        </div>
                    ) : (
                        <p className="text-sm text-stone-500">Wetterdaten sind derzeit nicht verfügbar.</p>
                    )}
                    <p className="text-[10px] text-stone-400 mt-2 text-right">Zuletzt aktualisiert: {weatherData?.lastUpdated || 'N/A'}</p>
                </div>
                {/* Ende des Wetter API Feeds */}

                <div className="space-y-3">
                    {[
                        // Thematische Platzhalterbilder für Strand- und Surf-Cams
                        { name: "Baleal (Almagreira) Cam", url: "https://www.surfline.com/surf-report/almagreira/5dea8356fe21a44513cecc3f", thumbnailUrl: "https://placehold.co/80x50/1C77B5/ffffff?text=WELLE" },
                        // Peniche Supertubos Platzhalter
                        { name: "Peniche (SuperTubos | Molhe Leste) Cam", url: "https://beachcam.meo.pt/livecams/peniche-supertubos/", thumbnailUrl: "https://placehold.co/80x50/3498db/ffffff?text=SUPERTUBOS" }, 
                        { name: "Baleal (Praia do Norte) Cam", url: "https://www.windy.com/de/-Webcams-Ferrel-Ferrel/webcams/1695285178?39.372,-9.339,16", thumbnailUrl: "https://placehold.co/80x50/F39C12/ffffff?text=STRAND" }, 
                        { name: "Baleal (Praia do Lagide) Cam", url: "https://gosurf.fr/webcam/fr/102/Peniche-Praia-do-Lagide", thumbnailUrl: "https://placehold.co/80x50/2ECC71/ffffff?text=LAGIDE" },
                    ].map((cam, index) => (
                        <a 
                            key={index} 
                            href={cam.url} 
                            target="_blank" 
                            rel="noopener noreferrer"
                            className="flex items-center p-4 bg-white rounded-xl shadow border border-stone-200 hover:shadow-cyan-100 transition-shadow duration-300"
                        >
                            {/* NEU: Thumbnail links */}
                            <div className="flex-shrink-0 mr-4 w-20 h-12 rounded overflow-hidden relative">
                                <img 
                                    src={cam.thumbnailUrl} 
                                    alt={`Vorschau von ${cam.name}`} 
                                    className="w-full h-full object-cover"
                                    // Fallback, falls die URL nicht lädt
                                    onError={(e) => { e.target.onerror = null; e.target.src="https://placehold.co/80x50/cccccc/333333?text=N%2FA" }}
                                />
                                {/* Kleines Play-Icon oder Indikator */}
                                <div className="absolute inset-0 flex items-center justify-center bg-black bg-opacity-20 text-white text-sm font-bold">▶</div>
                            </div>

                            {/* Text und Pfeil rechts */}
                            <div className="flex-1 min-w-0">
                                 <span className="text-lg font-medium text-cyan-700 block truncate">{cam.name}</span>
                            </div>
                            <svg className="w-5 h-5 text-cyan-500 ml-4 flex-shrink-0" fill="none" stroke="currentColor" viewBox="0 0 24 24" xmlns="http://www.w3.org/2000/svg"><path strokeLinecap="round" strokeLinejoin="round" strokeWidth="2" d="M10 6H6a2 2 0 00-2 2v10a2 2 0 002 2h10a2 2 0 002-2v-4M14 4h6m0 0v6m0-6L10 14"></path></svg>
                        </a>
                    ))}
                </div>
            </div>
        );
    };

    
    const renderAgentTab = () => (
        <div className="flex flex-col h-full bg-stone-50">
            <h2 className="text-2xl font-extrabold p-4 pb-2 text-cyan-700 border-b border-stone-200">🤖 FlowBot - KI-Assistent</h2>
            
            <div className="flex-1 overflow-y-auto p-4 space-y-3" style={{ maxHeight: 'calc(100vh - 200px)' }}>
                {agentChat.map((msg, index) => (
                    <div 
                        key={index} 
                        className={`flex ${msg.role === 'user' ? 'justify-end' : 'justify-start'}`}
                    >
                        <div className={`max-w-xs sm:max-w-sm p-3 rounded-xl shadow-md ${
                            msg.role === 'user' 
                                ? 'bg-cyan-400 text-cyan-900 rounded-br-none' 
                                : 'bg-white text-stone-800 rounded-tl-none border border-stone-200' 
                        }`}>
                            <span className={`block text-xs font-semibold mb-1 opacity-80 ${msg.role === 'user' ? 'text-cyan-800' : 'text-stone-500'}`}>
                                {msg.role === 'user' ? 'Sie' : 'FlowBot'}
                            </span>
                            <p className='whitespace-pre-wrap'>{msg.text}</p>
                            
                            {/* NEU: Quellenzitierung anzeigen */}
                            {msg.role === 'assistant' && msg.sources && msg.sources.length > 0 && (
                                <div className="mt-2 pt-2 border-t border-stone-100">
                                    <p className="text-xs font-semibold text-stone-500 mb-1">Quellen (Google Search):</p>
                                    <ul className="space-y-0.5">
                                        {msg.sources.map((source, srcIndex) => (
                                            <li key={srcIndex} className="text-[10px] text-stone-400 truncate">
                                                <a href={source.uri} target="_blank" rel="noopener noreferrer" className="hover:underline hover:text-cyan-600">
                                                    🔗 {source.title}
                                                </a>
                                            </li>
                                        ))}
                                    </ul>
                                </div>
                            )}

                        </div>
                    </div>
                ))}
                {agentIsLoading && (
                     <div className="flex justify-start">
                         <div className="bg-white p-3 rounded-xl shadow-md rounded-tl-none border border-stone-200">
                             <span className="block text-xs font-semibold mb-1 opacity-80">FlowBot</span>
                             <p className='text-sm text-cyan-600'>FlowBot sucht nach aktuellen Infos...</p>
                         </div>
                     </div>
                )}
                <div ref={chatEndRef} />
            </div>

            <form onSubmit={handleAgentSubmit} className="p-4 border-t border-stone-200 bg-white">
                <div className="flex space-x-2">
                    <input
                        type="text"
                        value={agentInput}
                        onChange={(e) => setAgentInput(e.target.value)}
                        placeholder="Fragen Sie FlowBot (z.b. 'Wo ist das beste WLAN?')..."
                        className="flex-1 p-3 border border-stone-300 rounded-xl focus:ring-cyan-500 focus:border-cyan-500"
                        disabled={agentIsLoading}
                    />
                    <button
                        type="submit"
                        className="px-4 py-3 bg-cyan-600 text-white rounded-xl font-semibold hover:bg-cyan-700 disabled:bg-cyan-300 transition-colors shadow-md shadow-cyan-300/50"
                        disabled={agentIsLoading || agentInput.trim() === ''}
                    >
                        Fragen
                    </button>
                </div>
            </form>
        </div>
    );

    const renderContent = () => {
        switch (activeTab) {
            case 'map':
                return renderMapTab();
            case 'events':
                return renderEventsTab();
            case 'cams':
                return renderCamsTab();
            case 'chat': // Wird zu Community
                return renderCommunityTab();
            case 'agent':
                return renderAgentTab();
            default:
                return null;
        }
    };

    // -----------------------------------------------------
    // 8. HAUPT LAYOUT
    // -----------------------------------------------------

    return (
        // HÖHEN-FIX: min-h-screen statt fester Höhe
        <div className="min-h-screen bg-stone-100 flex justify-center p-4">
            <div className="w-full max-w-lg bg-white rounded-2xl shadow-2xl flex flex-col min-h-[90vh]">
                <header className="p-4 border-b bg-cyan-600 rounded-t-2xl shadow-lg">
                    <h1 className="text-2xl font-black text-white tracking-wider">SILVER COAST NOMADS</h1>
                    <p className="text-sm text-cyan-100">Dein lokaler Hub für Work & Surf</p>
                </header>

                <main className="flex-1 overflow-y-auto">
                    {renderContent()}
                </main>

                {/* Navigation Bar */}
                <nav className="flex justify-around border-t border-stone-200 bg-white p-2 rounded-b-2xl shadow-lg">
                    <NavItem tab="map" current={activeTab} setTab={setActiveTab} icon="🗺️" label="Spots" />
                    <NavItem tab="events" current={activeTab} setTab={setActiveTab} icon="📅" label="Events" />
                    <NavItem tab="cams" current={activeTab} setTab={setActiveTab} icon="🌊" label="Surf Cams" />
                    <NavItem tab="chat" current={activeTab} setTab={setActiveTab} icon="💬" label="Community" /> {/* LABEL GEÄNDERT */}
                    <NavItem tab="agent" current={activeTab} setTab={setActiveTab} icon="🤖" label="FlowBot" />
                </nav>
            </div>
        </div>
    );
};

// Navigations-Komponente
const NavItem = ({ tab, current, setTab, icon, label }) => (
    <button
        onClick={() => setTab(tab)}
        className={`flex flex-col items-center p-2 rounded-xl transition-colors duration-200 ${
            current === tab ? 'text-cyan-700 bg-cyan-50 shadow-inner' : 'text-stone-500 hover:text-cyan-600 hover:bg-stone-100'
        }`}
    >
        <span className="text-xl">{icon}</span>
        <span className="text-xs mt-1 font-medium whitespace-nowrap">{label}</span>
    </button>
);

export default App;
