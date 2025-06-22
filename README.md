const fs = require('fs');
const path = require('path');
const Fuse = require('fuse.js');

class SlangTranslator {
    constructor() {
        this.slangData = this.loadSlangDatabase();
        this.setupFuzzySearch();
    }

    loadSlangDatabase() {
        const dataPath = path.join(__dirname, '../data/slang_database.json');
        return JSON.parse(fs.readFileSync(dataPath, 'utf8'));
    }

    setupFuzzySearch() {
        const slangTerms = Object.keys(this.slangData.slang_entries);
        this.fuse = new Fuse(slangTerms, {
            threshold: 0.3,
            includeScore: true
        });
    }

    detectGeneration(text) {
        const words = text.toLowerCase().split(/\s+/);
        const generations = ['gen_z', 'millennial', 'gen_x', 'boomer'];
        const scores = {};

        generations.forEach(gen => {
            scores[gen] = 0;
            const markers = this.slangData.generation_markers[gen];
            
            words.forEach(word => {
                if (markers.includes(word)) {
                    scores[gen] += 1;
                }
            });
        });

        const maxScore = Math.max(...Object.values(scores));
        if (maxScore === 0) return 'millennial';
        
        return Object.keys(scores).find(gen => scores[gen] === maxScore);
    }

    findSlangTerms(text) {
        const words = text.toLowerCase().split(/\s+/);
        const foundTerms = [];

        words.forEach(word => {
            if (this.slangData.slang_entries[word]) {
                foundTerms.push({
                    term: word,
                    match: 'exact'
                });
            }
        });

        return foundTerms;
    }

    translateSlang(text, targetGeneration = 'millennial', targetCountry = 'usa') {
        const sourceGeneration = this.detectGeneration(text);
        const slangTerms = this.findSlangTerms(text);
        
        if (slangTerms.length === 0) {
            return {
                translatedText: text,
                sourceGeneration,
                targetGeneration,
                targetCountry,
                translations: [],
                confidence: 0
            };
        }

        let translatedText = text;
        const translations = [];

        slangTerms.forEach(termData => {
            const term = termData.term;
            const slangEntry = this.slangData.slang_entries[term];
            
            if (slangEntry && slangEntry.definitions[targetGeneration] && 
                slangEntry.definitions[targetGeneration][targetCountry]) {
                
                const translation = slangEntry.definitions[targetGeneration][targetCountry];
                
                translatedText = translatedText.replace(
                    new RegExp(`\\b${term}\\b`, 'gi'), 
                    `**${translation}**`
                );
                
                translations.push({
                    original: term,
                    translation: translation,
                    chinese: slangEntry.chinese
                });
            }
        });

        return {
            translatedText,
            sourceGeneration,
            targetGeneration,
            targetCountry,
            translations,
            confidence: translations.length / slangTerms.length
        };
    }

    getAvailableOptions() {
        return {
            generations: ['gen_z', 'millennial', 'gen_x', 'boomer'],
            countries: ['usa', 'uk', 'australia', 'canada', 'south_africa', 'india']
        };
    }
}

module.exports = SlangTranslator;
