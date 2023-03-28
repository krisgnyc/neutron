# neutron
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <dirent.h>
#include <sys/wait.h>
#include <unistd.h>

#define MAX_PATH_LENGTH 1024

const char* vsts[] = {
    "Neutron 4 Compressor",
    "Neutron 4 Equalizer",
    "Neutron 4 Exciter",
    "Neutron 4 Gate",
    "Neutron 4 Sculptor",
    "Neutron 4 Transient Shaper",
    "Neutron 4 Unmask",
    "Neutron 4 Visual Mixer",
    "Neutron 4"
};

const char* aucomponents[] = {
    "iZNeutron4CompressorAUHook",
    "iZNeutron4EqualizerAUHook",
    "iZNeutron4ExciterAUHook",
    "iZNeutron4GateAUHook",
    "iZNeutron4SculptorAUHook",
    "iZNeutron4TransientShaperAUHook",
    "iZNeutron4UnmaskAUHook",
    "iZNeutron4VisualMixerAUHook",
    "iZNeutron4AUHook"
};

const char* bundles[] = {
    "iZNeutron4Compressor",
    "iZNeutron4Equalizer",
    "iZNeutron4Exciter",
    "iZNeutron4Gate",
    "iZNeutron4Sculptor",
    "iZNeutron4TransientShaper",
    "iZNeutron4Unmask",
    "iZNeutron4VisualMixer",
    "iZNeutron4"
};

void replace(char* file_path, const char* pattern, const char* replacement) {
    FILE* fp;
    char* line = NULL;
    size_t len = 0;
    ssize_t read;
    fp = fopen(file_path, "r+");
    if (fp == NULL) {
        perror("fopen");
        return;
    }
    while ((read = getline(&line, &len, fp)) != -1) {
        char* p = strstr(line, pattern);
        if (p != NULL) {
            size_t n = strlen(pattern);
            if (strlen(replacement) > n) {
                memmove(p + strlen(replacement), p + n, strlen(p) - n + 1);
            }
            memcpy(p, replacement, strlen(replacement));
            fseek(fp, -(read - strlen(line)), SEEK_CUR);
            fputs(line, fp);
        }
    }
    free(line);
    fclose(fp);
}

void prep(char* file_path) {
    pid_t pid = fork();
    if (pid == 0) {
        char* args[] = {"sudo", "xattr", "-cr", file_path, NULL};
        execvp(args[0], args);
        perror("execvp");
        exit(1);
    }
    else if (pid < 0) {
        perror("fork");
        return;
    }
    else {
        waitpid(pid, NULL, 0);
    }

    pid = fork();
    if (pid == 0) {
        char* args[] = {"sudo", "xattr", "-r", "-d", "com.apple.quarantine", file_path, NULL};
        execvp(args[0], args);
        perror("execvp");
        exit(1);
    }
    else if (pid < 0) {
        perror("fork");
        return;
    }
    else {
        waitpid(pid, NULL, 0);
    }

    pid = fork();
if (pid == 0) {
    char* args[] = {"sudo", "codesign", "--force", "--deep", "--sign", "-", file_path, NULL};
    execvp(args[0], args);
    perror("execvp");
    exit(1);
}
else if (pid < 0) {
    perror("fork");
    return;
}
else {
    waitpid(pid, NULL, 0);
}
void patch(char* file_path) {
replace(file_path, "\xFD\x7B\xBF\xA9\xFD\x03\x00\x91\x00\x10\x42\xF9(.{13})\x84\x4D", "\x20\x00\x80\x52\xC0\x03\x5F\xD6\x00\x10\x42\xF9${1}\x84\x4D"); //arm64-1
replace(file_path, "\xFD\x7B\xBF\xA9\xFD\x03\x00\x91\x00\x10\x42\xF9(.{13})\x64\x43", "\xA0\x00\x80\x52\xC0\x03\x5F\xD6\x00\x10\x42\xF9${1}\x64\x43"); //arm64-2
replace(file_path, "\x55\x48\x89\xE5\x48\x8B\xBF\x20\x04\x00\x00\x48\x8B\x35(.{12})\x61\x03", "\x31\xC0\xFF\xC0\xC3\x8B\xBF\x20\x04\x00\x00\x48\x8B\x35${1}\x61\x03"); //intel-1
replace(file_path, "\x55\x48\x89\xE5\x48\x8B\xBF\x20\x04\x00\x00\x48\x8B\x35(.{12})\x64\x03", "\xB8\x05\x00\x00\x00\xC3\xBF\x20\x04\x00\x00\x48\x8B\x35${1}\x64\x03"); //intel-2
}

int main(int argc, char** argv) {
char path[MAX_PATH_LENGTH];
int i, j;

// Patch VST3 plugins
for (i = 0; i < sizeof(vsts) / sizeof(char*); i++) {
    snprintf(path, MAX_PATH_LENGTH, "/Library/Audio/Plug-Ins/VST3/%s.vst3/Contents/Resources", vsts[i]);
    DIR* dir = opendir(path);
    if (dir == NULL) {
        continue;
    }

    struct dirent* entry;
    while ((entry = readdir(dir)) != NULL) {
        if (entry->d_type == DT_DIR && strcmp(entry->d_name, ".") != 0 && strcmp(entry->d_name, "..") != 0) {
            snprintf(path, MAX_PATH_LENGTH, "/Library/Audio/Plug-Ins/VST3/%s.vst3/Contents/Resources/%s/Contents/MacOS", vsts[i], entry->d_name);
            DIR* subdir = opendir(path);
            if (subdir != NULL) {
                struct dirent* subentry;
                while ((subentry = readdir(subdir)) != NULL) {
                    if (subentry->d_type == DT_REG && strcmp(subentry->d_name, bundles[i]) == 0) {
                        snprintf(path, MAX_PATH_LENGTH, "/Library/Audio/Plug-Ins/VST3/%s.vst3/Contents/Resources/%s/Contents/MacOS/%s", vsts[i], entry->d_name, subentry->d_name);
                        printf("Patching: %s\n", path);
                        patch(path);
                        prep(path);
                    }
                }
                closedir(subdir);
            }
        }
    }
    closedir(dir);
}

// Patch Audio Unit components
for (i = 0; i < sizeof(aucomponents) / sizeof(char*); i++) {
    snprintf(path, MAX_PATH_LENGTH, "/Library/Audio/Plug-Ins/Components/%s.component/Contents/Resources", aucomponents[i]);
    DIR* dir = opendir(path);
    if (dir == NULL) {
        continue;
    }

    struct dirent* entry;
    while ((entry = readdir(dir)) != NULL) {
        if (entry->d_type == DT_DIR && strcmp(entry->d_name, ".") != 0 && strcmp(entry->d_name, "..") != 0) {
            snprintf(path, MAX_PATH_LENGTH, "/Library/Audio/Plug-Ins/Components/%s.component/Contents/Resources/%s/Contents/MacOS", aucomponents[i], entry->d_name);
            DIR* subdir = opendir(path);
            if (subdir != NULL) {
                struct dirent* subentry;
                while ((subentry = readdir(subdir)) != NULL) {
                    if (subentry->d_type == DT_REG && strcmp(subentry->d_name, bundles[i]) == 0) {
                        snprintf(path, MAX_PATH_LENGTH, "/Library/Audio/Plug-Ins/Components/%s.component/Contents/Resources/%s/Contents/MacOS/%s", aucomponents[i], entry->d_name, subentry->d_name);
                        printf("Patching: %s\n", path);
                        patch(path);
                        prep(path);
                    }
                }
                closedir(subdir);
            }
        }
    }
    closedir(dir);
}

printf("Patching complete.\n");
return 0;

